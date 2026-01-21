# 输入脚本运行时规范

## 1. 定位与架构

### 1.1 系统定位
AndroidClient 是一个 **Input Runtime（宿主）**，负责：
- 采集原始输入数据（陀螺仪、触摸、手柄）
- 提供脚本执行环境
- 管理配置文件
- 确保输入链路的稳定性和可靠性

脚本是 **映射逻辑（可替换）**，负责：
- 将原始输入转换为游戏输入状态
- 实现自定义的输入处理算法
- 支持高度定制化的映射需求

### 1.2 架构图
```
RawInput → ScriptEngine.update(raw, state, event) → ScriptOutput → InputState → WebSocket
```

## 2. ABI 规范

### 2.1 脚本入口函数
脚本必须暴露唯一入口函数：
```javascript
function update(rawInput, inputState, event) {
    // 处理输入并返回结果
    return {
        frameId: rawInput.frameId,
        heldKeys: ["W", "A"],
        events: [],
        debug: {}
    };
}
```

### 2.2 RawInput 数据格式
```json
{
    "frameId": 123,
    "gyroPitch": 0.1,
    "gyroRoll": 0.2,
    "gyroYaw": 0.3,
    "touchPressed": false,
    "touchX": 0.5,
    "touchY": 0.5,
    "buttonA": false,
    "buttonB": true,
    "gamepad": {
        "axes": {
            "LX": 0.1,
            "LY": -0.2,
            "RX": 0.0,
            "RY": 0.0
        },
        "buttons": {
            "A": true,
            "B": false,
            "X": false,
            "Y": false
        }
    }
}
```

### 2.3 ScriptOutput 标准返回格式
```json
{
    "frameId": 123,
    "heldKeys": ["W", "A"],
    "events": [{"type":"tap","key":"SPACE"}],
    "debug": {"steer": 0.12}
}
```

## 3. 确定性要求

### 3.1 帧序号机制
- **Java 侧**为每次 `update()` 生成 `frameId`（单调递增）
- **JS 执行返回**时必须带回同一个 `frameId`
- **Java 侧**只接受“最新 frameId”的结果，旧结果直接丢弃
- **超时处理**：超过 5000ms 未返回结果则认定该帧失败

### 3.2 执行顺序保证
- 每帧 `update()` 必须严格按顺序执行
- 上一帧执行完成后才能开始下一帧
- 异步回调必须通过 frameId 过滤，确保结果顺序正确

## 4. 错误策略

### 4.1 错误类型
- **LOAD_ERROR**：脚本加载失败
- **COMPILE_ERROR**：脚本编译错误
- **RUNTIME_ERROR**：脚本执行时抛出异常
- **TIMEOUT**：脚本执行超时

### 4.2 错误处理机制
- **任意错误不得阻塞输入链路**
- 错误发生时自动触发回退机制：
  1. 尝试回滚到上一个配置文件
  2. 回滚失败则切换到传统 Pipeline
  3. 传统 Pipeline 也失败则使用安全映射（空输入）

### 4.3 错误日志
- 所有错误必须记录详细日志
- 包含错误类型、原因、脚本名称、帧序号
- 支持通过 WebSocket 发送调试信息

## 5. Profile 包与导入导出标准

### 5.1 Profile 包格式
使用 zip 压缩包，结构固定：
```
profile.zip/
  ├── profile.json      # 配置文件元数据
  ├── main.js           # 主脚本文件
  ├── lib/              # 可选的库文件目录
  │   └── utils.js
  ├── ui/               # 可选的 UI 配置目录
  │   └── layout.json
  └── assets/           # 可选的资源文件目录
      └── icon.png
```

### 5.2 profile.json Schema
```json
{
    "id": "uuid-v4",
    "name": "Gyro Steering",
    "version": "1.0.0",
    "author": "LineCat",
    "description": "Gyro-based steering with touch acceleration",
    "entry": "main.js",
    "engineApiVersion": "1.0.0",
    "compatibility": {
        "minApiLevel": 21
    }
}
```

### 5.3 校验策略
- **Schema 校验**：验证 profile.json 必填字段
- **文件存在性**：entry 文件必须存在且可读
- **函数暴露**：main.js 必须暴露 `update(raw, state, event)` 函数
- **版本兼容性**：检查 `engineApiVersion` 是否兼容

### 5.4 切换与回滚的原子性
- **原子切换**：新 profile 验证通过后才替换当前 profile
- **自动回滚**：加载失败/运行异常立即回滚到上一个 profile
- **状态保存**：保存当前和上一个 profile 的完整状态

## 6. 手柄支持

### 6.1 采集层
- **连接管理**：监听 `InputDevice` 的连接/断开事件
- **按键处理**：处理 `KeyEvent` 并映射到标准按键名称
- **轴处理**：处理 `MotionEvent` 并映射到标准轴名称
- **统一命名**：使用标准字符串 key 规则

### 6.2 标准按键和轴名称

#### 轴名称
| 标准名称 | 描述 |
|---------|------|
| LX | 左摇杆 X 轴 |
| LY | 左摇杆 Y 轴 |
| RX | 右摇杆 X 轴 |
| RY | 右摇杆 Y 轴 |
| LT | 左扳机 |
| RT | 右扳机 |
| DPadX | 方向键 X 轴 |
| DPadY | 方向键 Y 轴 |

#### 按键名称
| 标准名称 | 描述 |
|---------|------|
| A | A 键 |
| B | B 键 |
| X | X 键 |
| Y | Y 键 |
| L1 | 左肩键 |
| R1 | 右肩键 |
| L2 | 左扳机键 |
| R2 | 右扳机键 |
| Start | 开始键 |
| Select | 选择键 |
| L3 | 左摇杆按下 |
| R3 | 右摇杆按下 |
| Home | 主页键 |

### 6.3 脚本访问方式
脚本通过以下方式访问手柄数据：
```javascript
function update(rawInput) {
    const leftX = rawInput.gamepad.axes["LX"];
    const isAPressed = rawInput.gamepad.buttons["A"];
    // 处理输入...
}
```

## 7. 脚本测试工具

### 7.1 测试工具功能
- **输入**：一段 RawInput 序列（JSON 格式）
- **执行**：跑 N 帧 `update` 函数
- **输出**：每帧的 heldKeys/events
- **断言**：和期望输出比对

### 7.2 测试用例格式
```json
{
    "name": "Gyro Steering Test",
    "profile": "wmmt_gyro_profile",
    "inputSequence": [
        {"frameId": 1, "gyroRoll": 0.1},
        {"frameId": 2, "gyroRoll": -0.1}
    ],
    "expectedOutputs": [
        {"frameId": 1, "heldKeys": ["W", "A"]},
        {"frameId": 2, "heldKeys": ["W", "D"]}
    ]
}
```

## 8. 回退机制

### 8.1 开关控制
主链路只允许一个开关：
- `useScriptRuntime=true` 时走脚本运行时
- `useScriptRuntime=false` 时走旧 Pipeline（用于回退）

### 8.2 自动回退触发条件
- 脚本加载失败
- 脚本执行超时
- 脚本抛出未捕获异常
- 配置文件验证失败
- 运行时状态异常

## 9. Runtime 行为保证清单

### 9.1 帧顺序保证
- **Java 侧**为每次 `update()` 生成 `frameId`（单调递增）
- **JS 执行返回**时必须带回同一个 `frameId`
- **Java 侧**只接受“最新 frameId”的结果，旧结果直接丢弃
- 每帧 `update()` 必须严格按顺序执行
- 上一帧执行完成后才能开始下一帧

### 9.2 错误隔离保证
- **任意错误不得阻塞输入链路**
- 错误发生时自动触发回退机制
- 脚本异常不会影响 Java 侧稳定性
- 错误信息详细记录，便于调试

### 9.3 回退保证
- 回退到上一个可用 Profile
- 若上一个 Profile 不可用，回退到默认 Profile
- 若默认 Profile 不可用，切换到传统 Pipeline
- 回退失败时清空所有 heldKeys

### 9.4 Profile 切换保证
- 同步切换，阻塞当前线程
- 不允许在输入帧中途切换
- 切换成功后才替换当前 Profile
- 切换失败时清空所有 heldKeys

### 9.5 超时保证
- 每帧 `update()` 执行时间必须 < 16ms（60fps）
- 超过 5000ms 视为超时
- 超时后自动触发回退机制

## 10. ScriptContext API 分类

### 10.1 @Stable API（2年内不破坏）

#### 核心输入获取
- `RawInput getRawInput()` - 获取原始输入数据
- `float getAxis(String axisName)` - 获取摇杆轴值
- `boolean isGamepadButtonPressed(String buttonName)` - 检查游戏手柄按钮是否被按下

#### 核心输出控制
- `void holdKey(String key)` - 按下并保持键盘按键
- `void releaseKey(String key)` - 释放键盘按键
- `void releaseAllKeys()` - 释放所有键盘按键
- `boolean isKeyHeld(String key)` - 检查按键是否被按下

#### 数据访问辅助类
- `GyroData` - 提供更友好的陀螺仪数据访问接口
- `TouchData` - 提供更友好的触摸数据访问接口

### 10.2 @Experimental API（随时可能变）

#### 鼠标控制
- `void setMousePosition(float x, float y)` - 设置鼠标位置
- `void setMouseButton(String button, boolean pressed)` - 设置鼠标按键状态

#### 高级功能
- `GyroData getGyro()` - 获取陀螺仪数据（封装后的）
- `TouchData getTouch()` - 获取触摸数据（封装后的）

## 11. Profile 生命周期约定

### 11.1 Profile 切换
- **同步切换**：切换过程阻塞当前线程
- **禁止中途切换**：不允许在输入帧中途切换
- **原子替换**：切换成功后才替换当前 Profile
- **失败处理**：切换失败时清空所有 heldKeys

### 11.2 切换失败处理
- 回退到**上一个可用 Profile**
- 若上一个 Profile 不可用，回退到**默认 Profile**
- 若默认 Profile 不可用，切换到**传统 Pipeline**
- 最终回退失败时**清空所有 heldKeys**

### 11.3 Profile 卸载
- **强制释放所有按键**
- **清理相关资源**
- **确保状态一致性**
- 重置脚本引擎状态
- 清空当前和上一个 Profile 引用

### 11.4 自动回滚机制
- 当脚本引擎处于错误状态时自动回滚
- 回滚行为符合 `rollbackProfile()` 的约定
- 回滚失败时记录详细日志并清空按键

## 9. 性能要求

### 9.1 执行时间
- 每帧 `update()` 执行时间必须 < 16ms（60fps）
- 超过 5000ms 视为超时

### 9.2 内存占用
- WebView 内存占用必须 < 100MB
- 定期清理不再使用的资源

## 10. 安全要求

### 10.1 脚本沙箱
- 脚本只能访问指定的 API
- 禁止访问文件系统、网络等危险操作
- 严格限制脚本的权限

### 10.2 输入验证
- 所有输入必须经过验证
- 防止注入攻击
- 限制输入数据的范围

## 11. 版本兼容性

### 11.1 API 版本管理
- 使用 `engineApiVersion` 字段管理 API 版本
- 向后兼容策略：新版本必须支持旧版本的脚本
- 不兼容变更必须增加主版本号

### 11.2 回退机制
- 不兼容的脚本必须自动回滚到兼容版本
- 提供明确的错误信息

## 12. 调试与观测

### 12.1 日志系统
- 详细的日志记录
- 支持不同日志级别
- 包含脚本执行时间、帧序号等信息

### 12.2 调试面板
- 实时显示输入数据
- 显示脚本执行结果
- 支持查看历史记录
- 提供性能分析工具

### 12.3 远程调试
- 支持通过 WebSocket 发送调试信息
- 支持远程查看和修改脚本
- 支持远程测试和验证

## 13. 部署与更新

### 13.1 配置文件管理
- 支持从 assets 加载默认配置
- 支持从外部存储加载自定义配置
- 支持在线更新配置文件

### 13.2 脚本更新策略
- 增量更新：只更新修改的文件
- 原子更新：更新失败不影响当前运行
- 自动回滚：更新后运行异常自动回滚

## 14. 最佳实践

### 14.1 脚本编写
- 保持脚本简洁高效
- 避免复杂的计算
- 定期清理资源
- 使用模块化设计

### 14.2 测试策略
- 为每个功能编写测试用例
- 测试边界情况
- 测试性能和稳定性
- 定期运行回归测试

### 14.3 性能优化
- 减少不必要的计算
- 缓存计算结果
- 优化算法复杂度
- 避免频繁的 DOM 操作

## 15. 未来扩展

### 15.1 UI 自定义
- 支持脚本定义自定义 UI
- 支持动态加载 UI 组件
- 支持主题定制

### 15.2 多设备支持
- 支持多手柄同时连接
- 支持设备优先级管理
- 支持设备组合映射

### 15.3 高级算法支持
- 支持机器学习模型
- 支持复杂的输入预测
- 支持自适应映射

### 15.4 云同步
- 支持配置文件云同步
- 支持脚本云分享
- 支持社区脚本库

## 16. 术语表

| 术语 | 解释 |
|------|------|
| RawInput | 原始输入数据 |
| InputState | 处理后的输入状态 |
| ScriptEngine | 脚本执行引擎 |
| Profile | 配置文件 |
| ABI | 应用程序二进制接口 |
| FrameId | 帧序号 |
| HeldKeys | 当前按住的按键 |
| Events | 输入事件 |
| Safe Mapping | 安全映射（空输入）|  