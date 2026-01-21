# Test Goals for Input Script Runtime

This document defines the core test goals for the Input Script Runtime system. Each goal is written as a verifiable statement that can be proven true or false using test results.

## Test Goals

| 目标编号 | 用户视角目标 | 可观测证据 | 最小通过条件 | 
|---|---|---|---|
| G1 | 启动应用后运行时能启动 | runtime 事件序列 | `RUNTIME_STARTED`→`SCRIPT_ENGINE_READY` |
| G2 | WebSocket 可连接并发送 | ws 收包 | 3 秒内收到至少 1 帧合法消息 |
| G3 | Activity 销毁不影响 runtime | ws 继续收包/事件 | 关闭 Activity 后仍发送 |
| G4 | Profile 切换全释放 | ws 帧字段 | 切换后第一帧 heldKeys 为空 |
| G5 | 脚本崩溃自动回退 | runtime 事件+ws | `RUNTIME_ERROR` 后 `PROFILE_ROLLBACK` 且继续发帧 |

## Authoritative Observation Points

For each goal, a single most reliable observation point is used to prove the goal:

1. **G1**: Runtime events (highest priority)
2. **G2**: MockWebServer received messages
3. **G3**: MockWebServer received messages after Activity close
4. **G4**: WebSocket message content (heldKeys field)
5. **G5**: Runtime events + MockWebServer messages

## Pass/Fail Criteria

Each goal is considered **PASSED** if:
- The minimum passing condition is met
- The authoritative observation point provides clear evidence
- The test completes within the specified timeout

Each goal is considered **FAILED** if:
- The minimum passing condition is not met
- The authoritative observation point provides conflicting evidence
- The test exceeds the specified timeout

## Test Coverage

These goals cover the core functionality of the Input Script Runtime system, including:
- System startup
- WebSocket communication
- Runtime lifecycle management
- Profile switching
- Error recovery mechanisms

By verifying these goals, we can demonstrate that the system is functioning correctly from a user perspective.