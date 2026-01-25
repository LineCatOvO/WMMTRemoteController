# 测试技术设计（安卓环境，单测 + 集成测试）

本设计严格基于你已冻结的边界与两类结构：**PlatformAdaptationLayer.java** 负责 Android 接入与生存、输出 Raw；**InputAbstractionLayer.java** 负责归一化（仅 rotation 基准统一）、状态机、合并与输出 primitives。并采纳你对 0–1 归一化的纠正：抽象层输出 **基准方向下的像素坐标 + 明确坐标空间**，不做比例化/归一化。

---

## 测试目标与覆盖面

### 需要被证明的“系统级真相”

- **输入不会卡死**：任何中断（overlay 停止、焦点变化、权限撤销、窗口重建等）都能以 `CANCEL` 或等效终止收敛，抽象层 active pointers 归零。
- **坐标语义可解释**：抽象层输出的指针坐标是“**自然竖屏基准**下的像素坐标”，并携带 `CoordinateSpace(widthPx,heightPx,basis)`；不会被 clamp/缩放破坏语义。
- **陀螺仪语义忠实**：平台层只负责采集与透传；抽象层固定轴映射与单位（rad/s），采样率只影响密度，不进入语义。
- **背压可控且可观测**：Raw 队列满时只丢 sensor，且会产生 drop 事件；pointer 的关键帧（DOWN/UP/CANCEL）永不丢。
- **可回放与可复现**：同一份 raw 序列在 JVM 单测中回放，输出 primitives 序列稳定可比对（golden）。

---

## 测试分层与工具选型（安卓工程）

| 层级 | 目的 | 运行位置 | 工具建议 |
|---|---|---|---|
| JVM 单测 | 抽象层契约、状态机、旋转归一、合并、gyro 映射、回放一致性 | `test/` | JUnit4/5（任选）、Truth/AssertJ（任选） |
| Instrumentation 集成测试 | overlay/触摸/生命周期/服务/传感器注册注销/背压在真实 Android 上成立 | `androidTest/` | AndroidJUnit4 + ActivityScenario/ServiceTestRule（或等价） |
| 设备验证（可选） | 真机传感器可用性、不同 ROM 行为差异 | 手工 + 自动混合 | debug 构建加录制 |

> 由于你限制“只用两个类”，测试里允许写 **test-only helper**（例如 FakeSink、EventCollector、NDJSON parser），它们不进入生产架构，不算破坏约束。

---

## 需要冻结的数据契约（用于测试断言）

### 抽象层 Pointer primitive（建议冻结为像素坐标）

- `PointerFrame`
  - `timeNanos`
  - `pointersById: id -> {phase,xPx,yPx,pressure?}`
  - `changedIds`
  - `canceled`
  - `space: {widthPx,heightPx,basis=NATURAL_PORTRAIT}`

### Gyro primitive

- `GyroFrame`
  - `timeNanos`
  - `yawRate,pitchRate,rollRate`（单位 rad/s）
  - `accuracy`

### Raw 事件关键约束

- `RawPointerEvent(action=CANCEL)` 必须存在并能被抽象层收敛为 canceled frame
- Raw 队列背压时：只允许丢 `RawSensorEvent`，并发出 `RawDropEvent(kind=SENSOR)`

---

## 单元测试技术设计（JVM，主战场）

### 1) InputAbstractionLayer Pointer 状态机测试

#### 用例 IA-PTR-001：单指点击序列

- **输入**：RawWindowEvent(metrics=1080x2400, rotation=0)；DOWN(id0,x=100,y=200) → UP(id0,...)
- **断言**
  - 输出帧顺序：DOWN frame、UP frame
  - DOWN frame：`pointersById` 包含 id0，phase=DOWN
  - UP frame：phase=UP，且输出后 active 中不再存在 id0
  - `canceled=false`

#### 用例 IA-PTR-002：MOVE 合并不吞关键帧

- **输入**：DOWN → MOVE×N（密集）→ UP（夹在中间或尾部）
- **断言**
  - 输出帧数少于输入 MOVE 数（合并生效）
  - 但 UP 必须存在且在逻辑上晚于最后一次有效 MOVE
  - DOWN/UP 不被合并掉

#### 用例 IA-PTR-003：多指并发与 changedIds

- **输入**：id0 DOWN、id1 DOWN、交错 MOVE、分别 UP
- **断言**
  - pointersById 同时存在两指时，位置不串
  - changedIds 精确反映本帧变化的指针 id（至少对 DOWN/UP 必须精确）

#### 用例 IA-PTR-004：CANCEL 语义冻结

- **输入**：DOWN(id0) → MOVE → CANCEL
- **断言**
  - 输出中存在 `PointerFrame(canceled=true)`
  - canceled frame 后 active pointers 为空
  - CANCEL 不允许“隐式当 UP”而保留残留状态

---

### 2) InputAbstractionLayer 旋转归一化测试（像素坐标）

> 目标是证明：抽象层输出坐标是 **NATURAL_PORTRAIT 基准下的像素空间**，rotation 不会污染上层语义。

#### 用例 IA-ROT-001：rotation=0 直通

- **输入**：metrics(1080x2400,r0)，point(100,200)
- **断言**：输出 xPx=100,yPx=200，space=1080x2400

#### 用例 IA-ROT-002：rotation=90/180/270 映射正确

- **输入**：同一“屏幕视觉位置”的点，在不同 rotation 下给出对应 raw 坐标
- **断言**：归一到 NATURAL_PORTRAIT 后得到一致的 (xPx,yPx)

> 这组测试必须依赖你冻结的旋转变换公式。做法是：在测试里构造可验证的简单点（四角、中心、边中点），并断言映射后落在预期像素位置。

#### 用例 IA-ROT-003：metrics 变化与 space 更新

- **输入**：先 METRICS(1080x2400)，后 METRICS(1200x2400)；期间产生触摸
- **断言**：后续 PointerFrame.space.widthPx/heightPx 更新，且坐标转换使用新 metrics

---

### 3) InputAbstractionLayer Gyro 映射与精度测试

#### 用例 IA-GYR-001：轴映射冻结

- **输入**：RawSensorEvent(values=[a,b,c], accuracy=HIGH)
- **断言**：GyroFrame 的 yaw/pitch/roll 与你冻结的映射一致（例如 yaw=c, pitch=a, roll=b——以你最终定义为准）
- **补充断言**：符号方向固定（正负不被偷偷翻转）

#### 用例 IA-GYR-002：accuracy 透传且不门控

- **输入**：accuracy=LOW/UNRELIABLE
- **断言**：GyroFrame 仍输出，只是 accuracy 字段变化（除非你明确决定要 drop，此处按“忠实输出”冻结）

#### 用例 IA-GYR-003：timeNanos 单调性不被破坏

- **输入**：严格递增 timeNanos 的 raw
- **断言**：输出 GyroFrame timeNanos 不倒退（若你做合并/重采样，需要额外冻结规则；当前建议不重采样）

---

### 4) 回放一致性 Golden 测试（JVM）

#### 资产形式

- `src/test/resources/golden/*.ndjson`（raw events）
- `src/test/resources/golden_expected/*.ndjson`（expected primitives）

#### 用例 IA-RPL-001：pointer_tap.ndjson

- **断言**：回放 raw → 输出 primitives 与 expected 完全一致（浮点允许极小误差）

#### 用例 IA-RPL-002：pointer_cancel.ndjson

- **断言**：存在 canceled frame 且最终 active 为空

#### 用例 IA-RPL-003：gyro_axis_mapping.ndjson

- **断言**：轴映射每条都对

#### 用例 IA-RPL-004：rotation_and_touch.ndjson

- **断言**：space 与坐标符合基准方向定义

> Golden 的价值是“防未来改坏契约”。一旦你调整旋转公式或轴定义，golden 会立刻报警，逼你显式更新基线并写变更说明。

---

## 集成测试技术设计（Instrumentation，安卓真机/模拟器）

### 测试前提与约束

- overlay 权限（SYSTEM_ALERT_WINDOW）在自动化环境中**可能无法稳定自动授予**；设计为两类：
  - **CI 可跑**：不依赖真实 overlay 权限的部分（例如启动服务后内部状态、sensor 注册注销的替身路径）
  - **设备实验室/手工批准后自动跑**：真实 overlay + touch 注入

> 如果你必须在 CI 跑 overlay 触摸注入，需要你工程里提供 debug-only 的“测试 Activity/入口”，但这不要求新增生产类；可以通过 manifest 的 test-only 组件实现。

---

### 1) PlatformAdaptationLayer 生命周期与输出序列

#### 用例 PAL-INT-001：startOverlay 产生 ATTACHED + METRICS

- **步骤**
  1. 创建 `PlatformAdaptationLayer`，注入 `RawEventSink`（测试收集器）
  2. 调用 `startOverlay()`
- **断言**
  - 收到 `RawWindowEvent(kind=OVERLAY_ATTACHED)`
  - 在合理时间内（例如 1s）收到至少一次 `METRICS_CHANGED` 或携带 metrics 的 attached（取决于你实现）

#### 用例 PAL-INT-002：stopOverlay 产生 DETACHED，且传感器停止

- **步骤**：start → 等待 sensor 事件若干 → stop
- **断言**
  - 收到 DETACHED
  - stop 后不再有 RawSensorEvent（设定观察窗口，例如 500ms-1s）

---

### 2) Touch 端到端（overlay 场景）

> 这部分在自动化里最难，取决于你的 overlay 是否可被 UiAutomator/Instrumentation 注入触摸。若不可，至少保证 MotionEvent 流在 View 层触发（可通过测试 Activity 挂载同一 RootView 逻辑，但这会改变“必须 overlay”的真实性）。建议分两级：

#### 用例 PAL-TOUCH-001（强）：真实 overlay 触摸注入

- **步骤**：start overlay → 用 UiAutomator 点击 overlay 区域 → 滑动 → 抬起
- **断言**：RawPointerEvent 收到 DOWN/MOVE/UP

#### 用例 PAL-TOUCH-002（弱但 CI 友好）：同逻辑 View 的触摸回调可工作

- **步骤**：在测试 Activity 中创建与 overlay 同样的触摸入口（仍然走 PlatformAdaptationLayer 的 onTouch snapshot 逻辑）
- **断言**：RawPointerEvent 序列正确（尤其 CANCEL 处理、snapshot 复制）

---

### 3) CANCEL 端到端收敛测试

#### 用例 PAL-CANCEL-001：stopOverlay 必须导致 CANCEL 或等效终止

- **目标**：防止“按下态卡死”
- **步骤**
  1. 让系统产生 DOWN（注入或手工）
  2. 在未 UP 的情况下 stopOverlay
- **断言**
  - 平台层必须发出 RawPointerEvent(action=CANCEL) 或保证抽象层能收到终止信号
  - 将 raw 输给 InputAbstractionLayer 后，必须输出 `PointerFrame(canceled=true)` 且 active 清空

> 如果你实现选择是“平台层保证 CANCEL”，那此测试就锁死这条契约；反之若你选择“平台层不一定发 CANCEL，但 stopOverlay 会发一个特殊 WindowEvent 触发抽象层清空”，也必须写死并测试。当前基于你之前的契约，推荐“平台层必须发 CANCEL”。

---

### 4) Raw 背压与丢弃策略（Instrumentation）

#### 用例 PAL-BP-001：sensor 洪泛时只丢 sensor

- **步骤**
  1. start overlay + sensor capture
  2. 人为制造 sink 消费变慢（例如在 sink 回调里 sleep 或队列堆积）
- **断言**
  - 出现 RawDropEvent(kind=SENSOR)
  - 在洪泛期间注入 pointer DOWN/UP/CANCEL，仍能收到（关键帧不丢）

---

## 联合集成测试：PlatformAdaptationLayer → InputAbstractionLayer

> 这类测试验证“边界对齐”：平台层输出的 raw 能让抽象层输出正确 primitives。

#### 用例 E2E-001：真实输入 → primitives 不违背契约

- **步骤**
  1. 平台层 sink 直接把 raw 转发给抽象层
  2. 抽象层输出 sink 收集 PointerFrame/GyroFrame
- **断言**
  - 任意 CANCEL 发生后，PointerFrame canceled 且 active 清空
  - rotation 变化后，space 更新且输出坐标仍在 NATURAL_PORTRAIT 定义下可解释
  - gyro 输出 time 单调且轴映射正确

---

## 测试数据与可观测性设计

### 事件收集器（test-only）

- `RawEventCollector`：线程安全收集 raw 事件并支持 await 条件
- `PrimitiveCollector`：收集 PointerFrame/GyroFrame 并支持断言序列

### NDJSON 回放（test-only）

- 解析 `golden/*.ndjson` 生成 raw 对象
- 按文件顺序喂给 InputAbstractionLayer（无需真实时间）

---

## 通过标准（Definition of Done）

- **JVM 单测**：所有 IA-* 用例通过；golden 回放稳定
- **Instrumentation**：
  - PAL-INT-* 通过（至少生命周期 + sensor 注册注销 + 背压）
  - 若 overlay 注入可行：PAL-TOUCH-001、PAL-CANCEL-001 通过
- **无契约漂移**：任何旋转公式、轴映射、CANCEL 行为的更改都必须更新对应 golden 与用例说明

---

## 你需要我确认的两项“冻结输入”（否则 gyro 与 rotation 的断言无法写死）

1. **Gyro 轴映射**：yaw/pitch/rollRate 分别对应 `values[0..2]` 的哪一项？是否需要符号翻转？
2. **Rotation 变换公式**：NATURAL_PORTRAIT 基准下，rotation=90/180/270 时 `(xPx,yPx)` 的映射规则你采用哪一套（顺时针还是逆时针，以 Display rotation 为准）？

你把这两项写成明确规则后，我可以把上面所有用例进一步“落到可写代码”的粒度：给出每个测试用例的输入序列（raw）与期望输出序列（primitives），以及 golden NDJSON 的最小内容。