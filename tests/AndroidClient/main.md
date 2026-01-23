下面是**《安卓前端测试流程文档（Android Frontend Test Process v1.0）》**。  
这份文档**严格对齐你已经冻结的系统架构与执行端设计**，目标是：

> **让安卓前端在“输入采集 → 布局 → 状态生成 → 发送”这一整条链路上，  
> 可以被系统性、自动化、可回归地测试。**

文档定位是**测试流程与测试合同**，不是测试代码。

---

# 远程赛车输入控制系统
## 安卓前端测试流程文档
**Android Frontend Test Process v1.0**

---

## 1. 文档定位

本文档定义安卓前端（Android Client）的**完整测试流程**，用于指导：

- 功能测试
- 自动化测试脚本编写
- 回归测试
- 稳定性与异常测试

测试目标是验证安卓前端是否：

- 正确采集输入
- 正确执行布局与映射
- 正确生成 ControlResultState
- 正确发送状态
- 在异常情况下保持安全与一致性

---

## 2. 测试范围（冻结）

### 2.1 覆盖范围

| 模块 | 是否测试 |
|---|---|
| 触控输入采集 | ✔ |
| 物理按键输入 | ✔ |
| 传感器输入（如有） | ✔ |
| 布局加载与切换 | ✔ |
| LayoutEngine 执行 | ✔ |
| ControlResultState 生成 | ✔ |
| WebSocket 发送 | ✔ |
| 状态驱动模型 | ✔ |
| 禁用 / 清零行为 | ✔ |
| 异常与断连处理 | ✔ |

### 2.2 不在范围内

- 执行端输入注入行为
- 游戏响应
- 网络延迟质量（由系统测试覆盖）

---

## 3. 测试前置条件

### 3.1 设备与环境

- Android 10 及以上
- 至少一台真机（必须）
- 可选模拟器（仅用于 UI 测试）
- 网络环境：
    - 本地 Wi‑Fi
    - 可模拟断连

### 3.2 测试工具

- Android Instrumentation / Espresso
- UI Automator（多点触控）
- Mock WebSocket Server
- 日志采集工具（Logcat）
- 可选：屏幕录制用于人工验证

---

## 4. 安卓前端核心测试对象

### 4.1 核心数据对象

- `InputEvent`
- `Operation`
- `Layout`
- `LayoutEngine`
- `ControlResultState`

### 4.2 核心不变量（测试必须验证）

- ControlResultState 是**完整状态快照**
- 新状态覆盖旧状态
- 缺失字段等价于零状态
- 禁用控制时必须发送零状态
- UI 不存在时控制仍可运行（如后台）

---

## 5. 测试流程总览

测试流程分为 7 个阶段：

1. 启动与初始化测试
2. 输入采集测试
3. 布局加载与切换测试
4. LayoutEngine 执行测试
5. ControlResultState 生成测试
6. WebSocket 发送测试
7. 异常与安全测试

---

## 6. 测试流程（详细）

---

### 6.1 启动与初始化测试

#### 目标
验证应用启动后：

- 输入系统初始化完成
- 默认布局加载成功
- 状态为零状态

#### 测试步骤
1. 启动应用
2. 不进行任何输入
3. 观察日志与状态输出

#### 预期结果
- LayoutEngine 初始化完成
- ControlResultState = 零状态
- 未发送非零控制状态

---

### 6.2 输入采集测试

#### 目标
验证安卓前端能正确采集各种输入。

#### 测试用例

| 用例 | 输入 | 预期 |
|---|---|---|
| I1 | 单点触控 | 生成对应 Operation |
| I2 | 多点触控 | 多 Operation 同时存在 |
| I3 | 触控滑动 | 连续状态变化 |
| I4 | 物理按键 | 正确映射为 Operation |
| I5 | 输入释放 | Operation 结束 |

---

### 6.3 布局加载与切换测试

#### 目标
验证布局系统行为。

#### 测试步骤
1. 加载布局 A
2. 执行输入
3. 切换到布局 B
4. 继续输入

#### 预期结果
- 布局切换后：
    - 旧布局 Operation 不再生效
    - 新布局立即生效
- 切换瞬间发送零状态或新状态（按设计）

---

### 6.4 LayoutEngine 执行测试

#### 目标
验证 LayoutEngine 的核心逻辑。

#### 测试用例

| 用例 | 场景 | 预期 |
|---|---|---|
| L1 | 单 Operation | 正确生成控制结果 |
| L2 | 多 Operation | 按 zIndex 合并 |
| L3 | 冲突 Operation | 高优先级覆盖 |
| L4 | Operation 结束 | 状态回退 |
| L5 | 禁用 Layout | 输出零状态 |

---

### 6.5 ControlResultState 生成测试

#### 目标
验证生成的状态符合协议与执行端预期。

#### 核心检查点

- sequence 单调递增
- KeyboardState / GamepadState 字段合法
- 缺失字段等价于零
- 不包含 UI / Operation 信息

#### 测试步骤
1. 执行一系列输入
2. 捕获生成的 ControlResultState
3. 与预期快照比对

---

### 6.6 WebSocket 发送测试

#### 目标
验证状态发送行为。

#### 测试用例

| 用例 | 行为 | 预期 |
|---|---|---|
| W1 | 状态变化 | 立即发送 |
| W2 | 状态不变 | 不重复发送 |
| W3 | 禁用控制 | 发送零状态 |
| W4 | 断连 | 停止发送 |
| W5 | 重连 | 恢复发送 |

---

### 6.7 异常与安全测试

#### 目标
验证前端在异常情况下的安全性。

#### 测试用例

| 用例 | 场景 | 预期 |
|---|---|---|
| S1 | App 切后台 | 发送零状态 |
| S2 | App 被暂停 | 停止发送 |
| S3 | 网络断开 | 停止发送 |
| S4 | Layout 崩溃 | 自动清零 |
| S5 | 输入异常 | 不发送非法状态 |

---

## 7. 自动化测试脚本结构建议

```
androidTest/
  input/
    touchInputTest.kt
    keyInputTest.kt
  layout/
    layoutLoadTest.kt
    layoutSwitchTest.kt
  engine/
    layoutEngineTest.kt
    conflictTest.kt
  state/
    controlResultStateTest.kt
  network/
    websocketSendTest.kt
  safety/
    zeroStateTest.kt
```

---

## 8. 测试覆盖率要求（冻结）

- 输入采集覆盖率：100%
- LayoutEngine 分支覆盖率：100%
- ControlResultState 字段覆盖率：100%
- 异常路径覆盖率：100%

---

## 9. 文档状态

**Android Frontend Test Process v1.0 · 冻结版**

---

## 工程结论

> **安卓前端测试的核心不是 UI，  
> 而是“输入 → 状态 → 发送”的确定性。**

这份流程文档已经足以支撑：

- 自动化测试脚本编写
- 回归测试
- 与执行端的端到端联调

---

如果你愿意，下一步我可以继续输出：

- **ControlResultState 的前端快照断言模板**
- **LayoutEngine 的白盒测试用例**
- **安卓端 Mock WebSocket Server 实现**
- **端到端（Android → Executor）联调测试流程**

你现在已经拥有一套**完整、可验证、可维护的测试体系蓝图**。
