# 生命周期管理

## 1. 生命周期概述

Input Script Runtime 系统的生命周期管理涵盖了从应用启动到关闭的完整过程，包括各个组件的初始化、运行时状态管理和优雅关闭。良好的生命周期管理确保系统资源得到有效利用，各组件之间协同工作，并在各种情况下能够优雅地处理状态转换。

## 2. 组件层级结构

系统采用分层架构，各组件具有明确的父子关系和依赖顺序：

```
Application
└─ Runtime Service
   ├─ Input Manager
   ├─ Script Engine
   ├─ WebSocket Client
   └─ Profile Manager
      └─ Active Profile
         └─ JavaScript Script
```

## 3. 完整生命周期流程

### 3.1 应用启动阶段

```
1. 应用启动 (Application onCreate)
2. 检查权限 (网络、传感器等)
3. 初始化配置 (读取 config.json)
4. 启动 Runtime Service
5. Runtime Service 初始化
```

### 3.2 Runtime Service 初始化

```
1. Service onCreate() 回调
2. 初始化核心组件：
   a. Input Manager
   b. Profile Manager
   c. Script Engine
   d. WebSocket Client
3. 注册系统广播接收器 (电量变化、网络状态等)
4. 初始化传感器管理器
5. 加载默认 Profile
6. 建立 WebSocket 连接
7. 启动输入处理线程
```

### 3.3 Profile 加载与激活

```
1. Profile Manager 接收加载请求
2. 扫描 Profile 目录，获取可用 Profile 列表
3. 验证目标 Profile 的完整性和兼容性
4. 初始化 Script Engine 环境
5. 加载 JavaScript 脚本
6. 调用脚本的 init() 方法
7. 设置为活动 Profile
8. 触发 ProfileActivated 事件
```

### 3.4 运行时阶段

```
1. Input Manager 采集原始输入数据
2. 预处理输入数据 (滤波、归一化等)
3. 将预处理数据传递给 Script Engine
4. Script Engine 执行当前 Profile 的 update() 方法
5. 生成标准化 InputFrame
6. WebSocket Client 将 InputFrame 发送到后端
7. 更新系统状态和指标
8. 检查是否需要切换 Profile
9. 循环执行（约 60 次/秒）
```

### 3.5 Profile 切换

```
1. 接收 Profile 切换请求 (用户操作或自动切换)
2. 调用当前 Profile 的 cleanup() 方法
3. 释放当前 Profile 占用的资源
4. 触发 ProfileDeactivated 事件
5. 加载新的 Profile (执行 Profile 加载流程)
6. 更新活动 Profile 引用
```

### 3.6 Service 后台运行

```
1. Activity 进入后台
2. Service 转为后台运行模式
3. 降低输入采集频率 (可选)
4. 保持 WebSocket 连接活跃
5. 继续处理输入数据
6. 监听系统广播，处理特殊事件
```

### 3.7 Service 唤醒

```
1. 接收到系统唤醒广播
2. 或用户手动打开应用
3. 恢复正常输入采集频率
4. 更新系统状态
5. 触发 ServiceWakeup 事件
```

### 3.8 Service 关闭

```
1. 接收关闭请求 (用户操作或系统事件)
2. 停止输入处理线程
3. 断开 WebSocket 连接
4. 调用当前 Profile 的 cleanup() 方法
5. 释放所有组件资源：
   a. 关闭 Script Engine
   b. 停止传感器监听
   c. 释放 Input Manager 资源
   d. 关闭 WebSocket Client
6. 取消系统广播注册
7. 保存运行时状态 (可选)
8. Service onDestroy() 回调
```

### 3.9 应用终止

```
1. 所有 Activity 被销毁
2. Runtime Service 已关闭
3. 释放应用级资源
4. 应用进程终止
```

## 4. 状态转换图

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   INITIALIZING │────▶│    RUNNING     │────▶│   BACKGROUND   │
└────────────────┘     └────────────────┘     └────────────────┘
       ▲                       │                       │
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               ▼
                        ┌────────────────┐
                        │    STOPPED     │
                        └────────────────┘
```

### 4.1 状态说明

| 状态 | 描述 |
|------|------|
| **INITIALIZING** | 系统正在初始化，组件正在创建和配置 |
| **RUNNING** | 系统正常运行，处理输入数据并发送到后端 |
| **BACKGROUND** | 系统在后台运行，可能降低性能以节省资源 |
| **STOPPED** | 系统已停止，所有资源已释放 |

### 4.2 状态转换条件

| 转换 | 触发条件 |
|------|----------|
| INITIALIZING → RUNNING | 所有组件初始化完成，WebSocket 连接建立，Profile 加载成功 |
| RUNNING → BACKGROUND | 应用进入后台，Activity 被销毁 |
| BACKGROUND → RUNNING | 应用回到前台，或收到唤醒事件 |
| RUNNING → STOPPED | 收到关闭请求，或系统资源不足 |
| BACKGROUND → STOPPED | 收到关闭请求，或系统资源不足 |
| STOPPED → INITIALIZING | 系统重启 |

## 5. 关键生命周期事件

### 5.1 系统级事件

| 事件名称 | 描述 | 触发时机 |
|----------|------|----------|
| **ServiceStarted** | Runtime Service 启动完成 | Service onCreate() 执行完毕 |
| **ServiceStopped** | Runtime Service 已停止 | Service onDestroy() 执行完毕 |
| **ServiceWakeup** | Service 从后台唤醒 | 应用回到前台，或收到唤醒广播 |
| **ServiceBackgrounded** | Service 进入后台 | 应用进入后台 |

### 5.2 Profile 相关事件

| 事件名称 | 描述 | 触发时机 |
|----------|------|----------|
| **ProfileListUpdated** | 可用 Profile 列表更新 | 扫描 Profile 目录后 |
| **ProfileLoading** | 开始加载 Profile | 收到 Profile 加载请求 |
| **ProfileLoaded** | Profile 加载完成 | 脚本 init() 方法执行成功 |
| **ProfileActivated** | Profile 激活成功 | 设置为活动 Profile 后 |
| **ProfileDeactivated** | Profile 停用 | 调用 cleanup() 方法前 |
| **ProfileError** | Profile 操作错误 | 加载、验证或执行过程中出错 |
| **ProfileSwitched** | Profile 切换完成 | 新 Profile 激活后 |

### 5.3 WebSocket 事件

| 事件名称 | 描述 | 触发时机 |
|----------|------|----------|
| **WebSocketConnected** | WebSocket 连接建立 | 成功建立与后端的连接 |
| **WebSocketDisconnected** | WebSocket 连接断开 | 连接意外断开或主动关闭 |
| **WebSocketError** | WebSocket 错误 | 连接、发送或接收过程中出错 |
| **WebSocketReconnecting** | WebSocket 正在重连 | 开始尝试重新建立连接 |

### 5.4 输入处理事件

| 事件名称 | 描述 | 触发时机 |
|----------|------|----------|
| **InputStarted** | 输入处理开始 | 输入处理线程启动 |
| **InputStopped** | 输入处理停止 | 输入处理线程停止 |
| **InputFrameGenerated** | 生成 InputFrame | 脚本执行完成，生成标准化输入帧 |
| **InputFrameSent** | InputFrame 发送成功 | WebSocket 发送 InputFrame 成功 |
| **InputError** | 输入处理错误 | 采集或处理输入数据时出错 |

## 6. 资源管理

### 6.1 内存管理

- **组件初始化**：按需创建组件，避免过早分配大量内存
- **运行时**：定期清理不再使用的资源，如旧的输入帧数据
- **Profile 切换**：及时释放当前 Profile 占用的资源
- **后台运行**：减少内存使用，释放非必要资源
- **关闭阶段**：彻底释放所有资源，避免内存泄漏

### 6.2 CPU 管理

- **输入处理线程**：设置合理的优先级，避免占用过多 CPU
- **后台运行**：降低输入采集频率，减少脚本执行次数
- **低电量模式**：进一步降低 CPU 使用率，延长电池寿命
- **动态调整**：根据设备负载动态调整处理频率

### 6.3 传感器资源

- **按需启用**：只启用当前 Profile 需要的传感器
- **合理配置**：设置合适的传感器采样率
- **自动关闭**：在不需要时关闭传感器，节省电量
- **权限检查**：在使用前检查并请求传感器权限

## 7. 异常情况下的生命周期管理

### 7.1 应用崩溃

- **Service 自动重启**：通过 Service 的重启策略，在崩溃后自动重启
- **状态恢复**：重启后尝试恢复到崩溃前的状态
- **错误日志**：记录详细的崩溃信息，便于调试

### 7.2 设备重启

- **自启动**：如果用户允许，在设备重启后自动启动 Service
- **状态恢复**：加载上次保存的运行状态
- **重新连接**：重新建立 WebSocket 连接

### 7.3 网络变化

- **连接断开**：触发 WebSocketDisconnected 事件，开始重连
- **连接恢复**：成功重连后，触发 WebSocketConnected 事件
- **状态调整**：根据网络状态调整数据发送策略

### 7.4 电量变化

- **低电量警告**：在电量低于阈值时，触发低电量事件
- **省电模式**：自动切换到省电模式，降低性能以延长电池寿命
- **电量恢复**：电量恢复后，恢复正常运行模式

## 8. 生命周期管理最佳实践

### 8.1 对于应用开发者

- **合理启动 Service**：仅在需要时启动 Runtime Service
- **正确停止 Service**：在不需要时及时停止 Service，释放资源
- **处理配置变更**：妥善处理屏幕旋转等配置变更
- **监听系统事件**：及时响应系统事件，调整运行状态
- **测试边界情况**：测试各种异常情况，确保系统能够优雅处理

### 8.2 对于 Profile 开发者

- **实现完整的生命周期方法**：正确实现 init() 和 cleanup() 方法
- **资源管理**：在 cleanup() 方法中释放所有资源
- **状态保存**：在适当的时候保存和恢复状态
- **错误处理**：在生命周期方法中添加适当的错误处理
- **性能优化**：避免在 init() 方法中执行耗时操作

## 9. 监控与调试

### 9.1 生命周期日志

系统会记录关键的生命周期事件，便于监控和调试：

```
2026-01-21 14:30:00.123 INFO  [Service] Service started
2026-01-21 14:30:00.456 INFO  [ProfileManager] Loading profile: racing_standard
2026-01-21 14:30:00.789 INFO  [ScriptEngine] Script initialized successfully
2026-01-21 14:30:01.012 INFO  [WebSocket] Connected to ws://localhost:8080/ws/input
2026-01-21 14:30:01.234 INFO  [InputManager] Input processing started
2026-01-21 14:30:01.345 INFO  [Service] Runtime fully initialized, running
```

### 9.2 性能指标

系统监控以下与生命周期相关的指标：

| 指标 | 描述 | 单位 |
|------|------|------|
| **启动时间** | 从应用启动到 Runtime 运行的时间 | ms |
| **Profile 加载时间** | 加载一个 Profile 所需的时间 | ms |
| **Profile 切换时间** | 切换 Profile 所需的时间 | ms |
| **Service 运行时长** | Service 连续运行的时间 | s |
| **重启次数** | Service 自动重启的次数 | 次 |

## 10. 示例生命周期场景

### 10.1 正常启动场景

```
1. 用户打开应用
2. 应用检查并请求必要权限
3. 启动 Runtime Service
4. Runtime Service 初始化所有组件
5. 加载默认 Profile "racing_standard"
6. 建立 WebSocket 连接
7. 开始处理输入数据
8. 应用显示主界面，显示连接状态为 "已连接"
```

### 10.2 用户切换 Profile 场景

```
1. 用户在应用界面选择 "fps_aim_assist" Profile
2. 应用发送 Profile 切换请求给 Service
3. Service 调用当前 Profile 的 cleanup() 方法
4. Service 加载新的 Profile
5. 初始化新的脚本环境
6. 调用新脚本的 init() 方法
7. 设置为活动 Profile
8. 应用界面更新为新的 Profile 信息
```

### 10.3 应用进入后台场景

```
1. 用户按 Home 键，应用进入后台
2. 应用 Activity 进入 onPause() 和 onStop()
3. Service 收到系统广播，进入后台模式
4. 降低输入采集频率从 60 FPS 到 30 FPS
5. 保持 WebSocket 连接活跃
6. 继续处理输入数据，但降低性能
```

### 10.4 正常关闭场景

```
1. 用户在应用界面点击 "停止服务"
2. 应用发送停止请求给 Service
3. Service 停止输入处理线程
4. 断开 WebSocket 连接
5. 调用当前 Profile 的 cleanup() 方法
6. 释放所有组件资源
7. Service 进入 onDestroy()
8. 应用界面更新为 "服务已停止"
```

## 11. 结论

良好的生命周期管理是 Input Script Runtime 系统稳定运行的关键。通过明确的组件层级结构、完整的生命周期流程、合理的状态转换和资源管理，系统能够在各种情况下优雅地处理状态变化，确保资源得到有效利用，同时提供良好的用户体验。

开发者在设计和实现系统组件时，应该充分考虑生命周期管理，确保各组件能够正确初始化、运行和关闭，并且在异常情况下能够优雅地处理。同时，通过监控和日志记录，能够及时发现和解决生命周期管理中的问题，提高系统的可靠性和稳定性。