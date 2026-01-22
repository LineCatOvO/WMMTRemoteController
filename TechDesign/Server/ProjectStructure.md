下面是**在你明确选择“手柄通道为完整 Xbox（XInput）”之后的《执行端 Node.js 技术设计文档 · 完整修订版》**。  
这是一个**可直接作为实现合同的版本**，已经把：

- **完整 Xbox 手柄通道**
- **ViGEmBus + node-vigemclient**
- **枚举集合、映射边界、依赖冻结**

全部写死，不再留灰区。

---

# 远程赛车输入控制系统
## 执行端技术设计文档
**Executor Technical Design v1.3 · 完整修订版（Node.js / Windows）**

---

## 1. 文档定位

本文档定义远程赛车输入控制系统中**执行端（Executor）**的完整技术设计。

执行端是系统中的**实际输入层**，负责将前端发送的控制状态（ControlResultState）转换为**Windows 系统可见的真实键盘与真实手柄输入**。

本文档重点描述：

- 执行端运行环境与平台约束
- 键盘与手柄输出的**具体技术路线与依赖**
- **完整 Xbox（XInput）手柄通道定义与映射规则**
- Node.js Adapter 的实现边界与职责冻结

本文档**不定义**：

- 前端布局系统（UI / Operation / Mapping）
- 控制语义生成或冲突裁决
- 布局编辑器能力
- 通信协议字段细节（由通信协议文档定义）

---

## 2. 执行端角色与职责（冻结）

### 2.1 角色定义

执行端在系统中的角色是：

> **控制结果执行器（Control Result Executor）**

其职责是：

> 将前端生成的 **ControlResultState（控制状态快照）**  
> 映射为宿主操作系统可见的输入设备状态。

---

### 2.2 职责边界（冻结）

#### 执行端必须负责

- 接收并校验 ControlResultState
- 将控制状态映射为：
    - 键盘输入
    - **完整 Xbox（XInput）手柄输入**
- 维持状态驱动的持续输入语义
- 在断连、超时、异常时**独立清零**
- 返回 ACK 用于 RTT 与延迟观测

#### 执行端不得负责

- 不得理解 UI / Operation / Mapping 语义
- 不得修改控制状态内容
- 不得引入脚本或 DSL
- 不得决定控制是否启用
- 不得承担布局或语义逻辑

---

## 3. 平台与运行环境约束（冻结）

### 3.1 宿主平台

- **操作系统：Windows**
- **目标程序：Windows 游戏**
    - 可运行于 Wine / Proton
    - 输入必须落到 Windows 输入子系统

### 3.2 执行端运行时

- **Node.js**
- 允许使用 native addon
- 允许依赖系统级驱动（仅限手柄）

---

## 4. 输入与输出契约（执行端视角）

### 4.1 输入

- WebSocket 接收 ControlResultState
- ControlResultState 为**完整状态快照**
- 缺失 KeyboardState / GamepadState 等价于零状态

### 4.2 输出

- 输出为宿主系统可见的：
    - 键盘输入
    - **Xbox 360 控制器（XInput）输入**
- 输出语义为**状态维持**，不是事件播放

---

## 5. 手柄输出实现（完整 Xbox 通道 · 最终冻结）

### 5.1 技术路线选择（冻结）

| 项目 | 选择 |
|---|---|
| 虚拟手柄驱动 | **ViGEmBus** |
| Node.js 绑定库 | **`vigemclient`（node-ViGEmClient）** |
| 呈现设备类型 | **Xbox 360 控制器（XInput）** |
| 通道范围 | **完整 Xbox 通道** |

---

### 5.2 完整 Xbox（XInput）通道定义（冻结）

执行端支持并仅支持以下 **Xbox 360 控制器通道集合**：

#### 5.2.1 摇杆轴（Axes）

| 逻辑名 | XInput 通道 | 范围 |
|---|---|---|
| `LX` | Left Stick X | [-1.0, 1.0] |
| `LY` | Left Stick Y | [-1.0, 1.0] |
| `RX` | Right Stick X | [-1.0, 1.0] |
| `RY` | Right Stick Y | [-1.0, 1.0] |

---

#### 5.2.2 扳机（Triggers）

| 逻辑名 | XInput 通道 | 范围 |
|---|---|---|
| `LT` | Left Trigger | [0.0, 1.0] |
| `RT` | Right Trigger | [0.0, 1.0] |

---

#### 5.2.3 按钮（Buttons）

| 逻辑名 | XInput 按钮 |
|---|---|
| `A` | A |
| `B` | B |
| `X` | X |
| `Y` | Y |
| `LB` | Left Bumper |
| `RB` | Right Bumper |
| `BACK` | Back |
| `START` | Start |
| `LS` | Left Stick Press |
| `RS` | Right Stick Press |
| `DPAD_UP` | D-Pad Up |
| `DPAD_DOWN` | D-Pad Down |
| `DPAD_LEFT` | D-Pad Left |
| `DPAD_RIGHT` | D-Pad Right |

---

### 5.3 GamepadState → XInput 映射规则（冻结）

- 所有轴值：
    - 必须 clamp 到 [-1.0, 1.0]
- 所有扳机值：
    - 必须 clamp 到 [0.0, 1.0]
- 所有按钮：
    - true → 按下
    - false / 缺失 → 释放
- 缺失字段：
    - 等价于零状态（轴=0，扳机=0，按钮=false）

---

### 5.4 状态提交策略（冻结）

- 执行端**每个 ApplyScheduler tick**：
    - 构造完整 Xbox 状态
    - 通过 `vigemclient` **一次性提交整帧状态**
- 不允许：
    - 事件驱动提交
    - 局部字段更新
    - 依赖历史残留

---

### 5.5 依赖与部署约束（冻结）

- **必须安装 ViGEmBus 驱动**
- 驱动安装需要管理员权限
- `vigemclient` 为 native addon：
    - 需要 Windows Build Tools
- 执行端启动前必须验证驱动可用

---

## 6. 键盘输出实现（阶段性冻结方案）

### 6.1 技术路线选择（冻结）

> 当前阶段（Executor v1.x），键盘输出仅采用**用户态键盘输入注入方案**。

- 不引入虚拟 HID 键盘驱动
- 不实现内核态键盘设备

---

### 6.2 具体技术方案

- 使用 Windows 用户态输入注入 API
- Node.js 侧通过现成库调用

#### 推荐依赖（冻结）

- **`keysender`**

---

### 6.3 能力边界声明（冻结）

- 不保证所有游戏在 Raw Input / 反作弊环境下接受该输入
- 若目标程序拒绝：
    - 视为系统外部约束
    - 不构成执行端缺陷

---

## 7. Node.js Adapter 实现边界（冻结）

### 7.1 KeyboardAdapter

- 仅负责：
    - KeyDown / KeyUp 调用
- 不得：
    - 保存跨帧状态
    - 引入时间逻辑
    - 决定清零

---

### 7.2 GamepadAdapter（Xbox）

- 仅负责：
    - 将完整 Xbox 状态写入虚拟控制器
- 不得：
    - 插值、平滑
    - 局部更新
    - 状态缓存决策

---

## 8. 模块与依赖总结

### 8.1 Node.js 依赖

| 功能 | 依赖 |
|---|---|
| WebSocket | ws |
| 键盘输出 | keysender |
| 手柄输出 | vigemclient |
| 调度 | Node.js timer / hrtime |

### 8.2 系统级依赖

- ViGEmBus 驱动
- Windows 输入子系统

---

## 9. 当前阶段冻结结论

执行端在 v1.x 阶段被明确冻结为：

- **实际输入映射层**
- **完整 Xbox（XInput）手柄支持**
- **键盘为阶段性用户态注入**
- **安全回退优先于输入完整性**

---

## 10. 文档状态

**Executor Technical Design v1.3 · 冻结版**

---

### 工程结论（最终）

> **执行端现在具备：**
>
> - 明确的输入设备层级
> - 明确的技术依赖
> - 明确的 Xbox 通道全集
> - 明确的阶段性与非目标能力
>
> 可以直接进入实现与测试阶段。

---

如果你愿意，下一步我可以直接：

- 输出 **`mapping.ts` 的完整 Xbox 枚举 → ViGEm 字段映射表**
- 或给出 **`vigemclient` 的完整可运行 Adapter 实现**
- 或写 **执行端集成测试用例（验证每个 Xbox 通道）**

你现在这套执行端设计，已经是**工业级清晰度**了。
