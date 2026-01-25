# 远程赛车输入控制系统
**技术设计文档（Technical Design v1.1.2 · 通信层与数据交互规范 · 完整修订版）**

---

## 1. 文档定位

本文档定义客户端与执行端之间的**通信层（Transport）与数据交互（Data Interaction）合同**，用于约束：

- WebSocket 连接模型与角色职责
- 控制数据的双通道发送模型（状态通道 + 事件通道）
- ACK 语义、丢包统计、延迟测量与告警
- Config 控制平面可靠更新（独立 ACK、重发、前端呈现语义）
- 执行端超时清零与零输出语义
- 错误与异常的消息语义与不变量

本文档不定义：编码格式（JSON/CBOR/Protobuf）、具体字段名大小写、类/模块实现。

---

## 2. 通信模型总览

### 2.1 通信拓扑与约束

- **通信方式：** WebSocket
- **连接方向：** 客户端主动连接，执行端被动监听
- **连接数量：** 单连接（单会话）
- **运行范围：** 单设备本地（localhost）

---

### 2.2 通信角色与职责边界

| 角色 | 责任 | 不负责 |
|---|---|---|
| **客户端 Client** | 发送控制数据（状态/事件）、发送配置、计算延迟与丢包率、展示告警 | 输入注入、执行端内部状态管理 |
| **执行端 Server** | 接收并应用控制数据、返回 ACK、执行注入、超时清零、返回状态与错误 | 解析布局、理解输入源、UI/编辑器语义 |

---

## 3. 消息类型与互斥规则

### 3.1 消息类型总览

| 类型 | 方向 | 语义 |
|---|---|---|
| **State** | Client → Server | 完整控制状态快照（状态通道） |
| **Event** | Client → Server | 相对变化（事件通道） |
| **StateAck** | Server → Client | 状态通道确认（区间确认） |
| **EventAck** | Server → Client | 事件通道确认（可选，默认可关闭） |
| **Config** | Client → Server | 配置更新（控制平面） |
| **ConfigAck** | Server → Client | 配置确认（必须） |
| **Status** | Server → Client | 执行端状态快照（观测面） |
| **Error** | 双向 | 错误与异常语义 |

---

### 3.2 消息互斥不变量（必须）

- **任何单条消息不得同时包含：**
    - State 与 Config
    - Event 与 Config
    - State 与 Event

即：**控制平面（Config）与数据平面（State/Event）严格分离**。

---

## 4. 双通道发送模型

### 4.1 模型定义

系统支持两条可独立开关的数据通道：

| 通道 | 发送内容 | 目标 |
|---|---|---|
| **状态通道 State Channel** | **完整状态快照**（State） | 自洽、可恢复、强稳定 |
| **事件通道 Event Channel** | **相对变化**（Event） | 低延迟、低带宽、快速响应 |

两通道可同时启用，或单独启用，但**至少启用一种**。

---

### 4.2 用户调配语义（必须）

| 调配倾向 | 结果语义 |
|---|---|
| **事件多、状态少** | 响应更快、网络负担更小、对丢包更敏感 |
| **状态多、事件少** | 稳定性更高、恢复能力更强、开销更高 |

告警与观测仅提示，不自动替用户改变策略。

---

## 5. 状态通道 State 规范

### 5.1 State 语义

State 表示：

> **“在此时间点执行端应当执行的完整控制状态”**

它是**状态快照**，不是事件。

---

### 5.2 State 必备语义字段

| 字段 | 语义 |
|---|---|
| **stateId** | 客户端生成的单调递增标识（会话内） |
| **clientSendTs** | 客户端发送时间戳（用于 RTT 延迟测量） |
| **keyboardState** | 键盘完整状态 |
| **gamepadState** | 手柄完整状态 |
| **flags** | 包含 zero-output 等标志 |

---

### 5.3 State 信息类型详细定义

#### 5.3.1 基础结构

```json
{
  "stateId": 12345,
  "clientSendTs": 1769340764376,
  "keyboardState": [...],
  "gamepadState": {
    "buttons": [...],
    "joysticks": {...},
    "triggers": {...}
  },
  "flags": ["zero-output"]
}
```

#### 5.3.2 keyboardState 字段定义

| 字段 | 类型 | 语义 |
|---|---|---|
| **keyboardState** | Array<KeyboardEvent> | 键盘所有按键事件的数组 |

**KeyboardEvent 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **keyId** | String | 键位标识符 | 如 "KEY_W", "KEY_A", "KEY_S", "KEY_D" 等 |
| **eventType** | String | 事件类型 | "pressed", "released", "held" |

#### 5.3.3 gamepadState 字段定义

**gamepadState 结构**：

| 字段 | 类型 | 语义 |
|---|---|---|
| **buttons** | Array<GamepadButtonEvent> | 手柄按键事件数组 |
| **joysticks** | Object | 摇杆状态字典 |
| **triggers** | Object | 扳机状态字典 |

**GamepadButtonEvent 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **buttonId** | String | 按键标识符 | 如 "BUTTON_A", "BUTTON_B", "BUTTON_X", "BUTTON_Y" 等 |
| **eventType** | String | 事件类型 | "pressed", "released", "held" |

**joysticks 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **left** | Object | 左摇杆状态 | {"x": -1.0 到 1.0, "y": -1.0 到 1.0} |
| **right** | Object | 右摇杆状态 | {"x": -1.0 到 1.0, "y": -1.0 到 1.0} |
| **deadzone** | Number | 死区阈值 | 0.0 到 1.0 |

**triggers 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **left** | Number | 左扳机值 | 0.0 到 1.0 |
| **right** | Number | 右扳机值 | 0.0 到 1.0 |

#### 5.3.4 flags 字段定义

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **flags** | Array<String> | 标志数组 | "zero-output", "safe-mode", "debug-enabled" 等 |

---

### 5.4 覆盖原则（必须）

- 新的 State 必须能够**完全覆盖并替代**旧状态
- 执行端以“最后接收的有效 State”为权威基准

---

### 5.5 状态同步频率

- 状态通道支持按配置频率同步（频率由客户端设置）
- 频率配置属于客户端本地发送策略，不要求服务端参与决策

---

## 6. 事件通道 Event 规范

### 6.1 Event 语义

Event 表示：

> **“相对于某个已知基准状态，发生了哪些控制变化”**

它**不得携带完整控制状态**。

---

### 6.2 Event 必备语义字段

| 字段 | 语义 |
|---|---|
| **eventId** | 客户端生成的单调递增标识（会话内） |
| **baseStateId** | 事件所依附的基准状态（必须是客户端认为已生效的状态） |
| **clientSendTs** | 客户端发送时间戳（用于 RTT 延迟测量） |
| **delta** | 相对变化内容（仅变化项） |
| **flags** | 可含 zero-output 请求（见零输出语义） |

---

### 6.3 Event 信息类型详细定义

#### 6.3.1 基础结构

```json
{
  "eventId": 67890,
  "baseStateId": 12345,
  "clientSendTs": 1769340764376,
  "delta": {
    "keyboard": [...],
    "gamepad": {
      "buttons": [...],
      "joysticks": {...},
      "triggers": {...}
    }
  },
  "flags": []
}
```

#### 6.3.2 delta.keyboard 字段定义

| 字段 | 类型 | 语义 |
|---|---|---|
| **delta.keyboard** | Array<KeyboardEventDelta> | 键盘按键变化事件的数组 |

**KeyboardEventDelta 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **keyId** | String | 键位标识符 | 如 "KEY_W", "KEY_A", "KEY_S", "KEY_D" 等 |
| **eventType** | String | 事件类型 | "pressed", "released" |

#### 6.3.3 delta.gamepad 字段定义

**delta.gamepad 结构**：

| 字段 | 类型 | 语义 |
|---|---|---|
| **buttons** | Array<GamepadButtonEventDelta> | 手柄按键变化事件数组 |
| **joysticks** | Object | 摇杆变化状态字典 |
| **triggers** | Object | 扳机变化状态字典 |

**GamepadButtonEventDelta 结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **buttonId** | String | 按键标识符 | 如 "BUTTON_A", "BUTTON_B", "BUTTON_X", "BUTTON_Y" 等 |
| **eventType** | String | 事件类型 | "pressed", "released" |

**joysticks 变化结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **left** | Object | 左摇杆变化状态 | {"x": -1.0 到 1.0, "y": -1.0 到 1.0} |
| **right** | Object | 右摇杆变化状态 | {"x": -1.0 到 1.0, "y": -1.0 到 1.0} |

**triggers 变化结构**：

| 字段 | 类型 | 语义 | 取值范围 |
|---|---|---|---|
| **left** | Number | 左扳机变化值 | 0.0 到 1.0 |
| **right** | Number | 右扳机变化值 | 0.0 到 1.0 |

---

### 6.4 事件消息约束（必须）

1. **非自洽：** Event 不构成完整状态
2. **依附性：** Event 必须绑定 baseStateId
3. **可丢弃：** Event 允许被丢弃且不要求重传
4. **不可补偿：** 执行端不得“补事件/重放事件”

---

### 6.5 事件应用规则（必须）

执行端处理 Event 时：

- 若 `baseStateId` 与执行端当前权威状态不匹配（或不再可用），则
    - **直接丢弃该 Event**
    - 并通过 Status 或 Error 暴露“事件被丢弃”的原因（用于观测）

这保证：事件丢失不会导致执行端进入不一致状态。

---

## 7. 零输出 Zero Output 语义

### 7.1 零输出定义

零输出等价于：

- 键盘：无按键按下
- 手柄：所有按键 false，所有轴为 0，扳机为 0

---

### 7.2 零输出的传输方式

- **首选：通过 State 表达零输出**（flags 标记 zero-output）
- 事件通道可携带 “请求清零” 的事件，但执行端最终必须落到“权威状态为零”。

---

### 7.3 必须触发零输出的场景

| 场景 | 责任方 | 要求 |
|---|---|---|
| 客户端禁用输出 | Client | 立即发送零输出（State） |
| 布局切换开始 | Client | 先清零再切换 |
| 安全切断 | Client/Server | 立即零输出并可观测 |
| 连接断开/客户端崩溃 | Server | 超时触发清零（见 §10） |

---

## 8. ACK 机制与丢包统计

### 8.1 StateAck 语义（区间确认）

StateAck 表示：

> **“stateId 及之前的所有状态已被覆盖或执行。”**

即：StateAck 是**累计确认（cumulative ACK）**。

---

### 8.2 StateAck 必备语义字段

| 字段 | 语义 |
|---|---|
| **ackStateId** | 累计确认到的最大 stateId |
| **serverRecvTs** | 执行端收到该状态的时间（观测用途） |
| **serverApplyTs** | 执行端应用/执行时间（观测用途） |
| **status** | success / rejected（含拒绝原因） |

---

### 8.3 跳包与丢包事件定义（必须）

若客户端观察到 ACK 前进出现“跳跃”：

- 设上一次确认为 `lastAckStateId`
- 本次确认为 `ackStateId`

当满足：

\[
ackStateId > lastAckStateId + 1
\]

则视为发生 **丢包事件（Dropped State Event）**，被丢弃的状态数量为：

\[
droppedCount = ackStateId - lastAckStateId - 1
\]

> **注意：**这里“丢包”语义是**状态未被执行而被覆盖**，不等同于网络层丢包。

---

### 8.4 丢包率指标（客户端必须提供）

客户端应统计并展示**丢包率（Dropped State Rate）**，至少支持：

- **窗口统计：**最近 N 次 State 的丢包比例，或最近 T 秒的丢包比例
- **展示语义：**作为稳定性指标呈现给用户（不触发自动行为）

---

### 8.5 EventAck（可选）

事件通道的 ACK 为可选能力：

- 默认可以关闭（以降低开销）
- 若启用，EventAck 语义仅用于观测，不用于可靠性保证

---

## 9. 延迟测量与告警

### 9.1 延迟测量必须使用 RTT（修订后的不变量）

由于客户端与执行端时间基准不保证一致，延迟必须基于 RTT 估计：

- 客户端记录发送时刻：`clientSendTs`
- 客户端记录收到 ACK 时刻：`clientAckRecvTs`

估计延迟（RTT）：

\[
rtt = clientAckRecvTs - clientSendTs
\]

> serverRecvTs / serverApplyTs 仅用于观测与诊断，不用于 RTT 计算。

---

### 9.2 告警阈值

| RTT | 告警级别 | 语义 |
|---|---|---|
| ≤ 50ms | 正常 | 控制体验正常 |
| > 50ms | 黄色警告 | 延迟可感知 |
| > 100ms | 红色严重警告 | 显著影响体验 |

告警仅用于提示，不自动调整发送策略。

---

## 10. 执行端超时清零

### 10.1 超时清零语义

执行端必须维护接收超时：

- 若超过 `timeoutMs` 未收到新的 **State**（或满足等价“可维持活性”的条件），则
    - 执行端必须进入 **零输出**
    - 并通过 Status 明确报告“已清零 + 原因=timeout”

---

### 10.2 timeoutMs 配置

- `timeoutMs` 为可配置参数
- 默认值：500ms
- 客户端可通过 Config 更新该值（见 §11）

---

## 11. Config 控制平面规范（可靠更新）

### 11.1 Config 语义

Config 用于更新执行端控制平面参数（例如 timeoutMs、ackEnabled 等）。它**不携带控制状态**，并且必须可靠确认。

---

### 11.2 Config 必备语义字段

| 字段 | 语义 |
|---|---|
| **configId** | 客户端生成的单调递增标识（会话内） |
| **clientSendTs** | 客户端发送时间（用于超时重发逻辑） |
| **changes** | 配置变更集合（键值对语义） |

---

### 11.3 ConfigAck 必备语义字段（必须）

| 字段 | 语义 |
|---|---|
| **ackConfigId** | 对应的 configId |
| **status** | applied / rejected |
| **errorDetail** | 若 rejected，提供原因 |

---

### 11.4 Config 与 ControlState 的分离（重申）

- Config 与 State/Event **不得同包**
- Config 的生效不需要与某条 State 绑定；**完成时刻不做硬限制**

---

### 11.5 Config 完成时间与用户体验语义

- 工程期望：配置更新应尽快完成，通常应不超过 1 秒
- 若超过期望时间仍未收到 ConfigAck：
    - 客户端应显示“配置更新中”状态
- 若重发多次仍失败：
    - 客户端应展示错误内容（由 ConfigAck 的 rejected 或超时失败给出）

---

### 11.6 Config 重发规则（必须）

- 若客户端在合理等待时间内未收到 ConfigAck：
    - 必须重发同一 configId 的 Config（或以等价方式保证幂等）
- 执行端必须将 Config 视为幂等更新：
    - 重复收到同一 configId 不得产生非预期副作用

---

## 12. Status 观测面规范

### 12.1 Status 语义

Status 用于执行端提供状态快照，用于诊断与 UI 展示，至少包含：

- 当前是否处于零输出
- 最近一次清零原因（timeout / disable / security / error）
- 当前 ackStateId（累计确认进度）
- 事件丢弃统计（可选，但建议）

---

### 12.2 Status 发送策略

- 可周期性发送，或在状态变化时发送
- Status 不要求 ACK

---

## 13. Error 规范

### 13.1 Error 语义字段（建议但强烈推荐）

| 字段 | 语义 |
|---|---|
| **type** | ProtocolError / ExecutionError / SecurityInterrupt / ConfigError |
| **severity** | warning / recoverable / fatal |
| **requiresZero** | 是否必须进入零输出 |
| **detail** | 人类可读错误信息 |

---

### 13.2 Error 行为约束（必须）

- 任何 fatal 或 requiresZero 的错误必须导致执行端进入零输出
- Error 不得导致“未知状态继续注入”

---

## 14. 通信层不变量总结（必须遵守）

1. **控制平面与数据平面分离不变量**：Config 不与 State/Event 同包
2. **状态覆盖不变量**：State 完全覆盖旧状态
3. **事件相对不变量**：Event 只描述变化、可丢弃、不可补偿
4. **累计确认不变量**：StateAck 为区间确认（含覆盖/执行语义）
5. **丢包可观测不变量**：ACK 跳跃必须记录为丢包事件并形成丢包率
6. **RTT 延迟不变量**：延迟基于 RTT，不使用跨进程时间差
7. **超时自清零不变量**：执行端不得依赖客户端主动清零
8. **零输出优先不变量**：不确定状态下优先零输出

---

## 15. 与功能文档的映射说明

| 功能文档条款 | 本文对应章节 |
|---|---|
| 事件/状态发送可开关、频率可配 | §4、§5.4 |
| ACK 确认与延迟告警 | §8、§9 |
| 丢包率显示 | §8.3–§8.4 |
| Config 可靠更新 + 前端更新中/错误 | §11 |
| 执行端超时清零 + 默认 500ms | §10 |

---

## 16. 下一步建议

为了进入可实现阶段，下一份最硬的文档应是：

- **技术设计 v1.2：客户端发送调度器与融合逻辑**
    - **状态通道**发送频率与策略
    - **事件通道**生成规则（哪些变化发事件、如何绑定 baseStateId）
    - 丢包率与 RTT 的采样窗口与统计方式（语义级）
    - Config 更新流程（重发、并发、互斥）

如果你同意，我可以直接输出 v1.2，并保证它完全遵守本通信合同。
