# Test Matrix for Input Script Runtime

This document maps test goals to specific E2E tests, establishing clear traceability between what we want to prove and which tests are responsible for proving it.

## Test Goal to Test Mapping

| 目标编号 | 覆盖测试 | 证明方式 | 
|---|---|---|
| G1 | `AppLaunchE2E` | await 事件序列 |
| G2 | `AppLaunchE2E` | MockWsServer 收到 frame |
| G3 | `RuntimeServiceE2E` | Activity close 后仍收包 |
| G4 | `ProfileSwitchE2E` | 切换后第一帧 heldKeys 为空 |
| G5 | `ScriptFailureRollbackE2E` | error→rollback→继续收包 |

## Test Coverage Requirements

- **Minimum Coverage**: Each goal must be covered by at least 1 test
- **Critical Coverage**: Key goals (G1, G2, G5) should have at least 2 tests with different observation points
- **Cross Validation**: Whenever possible, use multiple observation points to verify the same goal

## Test Details

### AppLaunchE2E
- **Goal Coverage**: G1, G2
- **Observation Points**: Runtime events, WebSocket messages
- **Pass/Fail Criteria**:
  - G1: Receives expected startup events in order
  - G2: Receives at least 1 valid WebSocket frame

### RuntimeServiceE2E
- **Goal Coverage**: G3
- **Observation Points**: WebSocket messages after Activity close
- **Pass/Fail Criteria**:
  - G3: Continues sending WebSocket frames after Activity is destroyed

### ProfileSwitchE2E
- **Goal Coverage**: G4
- **Observation Points**: WebSocket message content
- **Pass/Fail Criteria**:
  - G4: First frame after profile switch has empty heldKeys

### ScriptFailureRollbackE2E
- **Goal Coverage**: G5
- **Observation Points**: Runtime events, WebSocket messages
- **Pass/Fail Criteria**:
  - G5: Receives RUNTIME_ERROR → PROFILE_ROLLBACK events and continues sending frames

### WebSocketReconnectE2E
- **Goal Coverage**: Additional coverage for WebSocket reliability
- **Observation Points**: Runtime events, WebSocket messages
- **Pass/Fail Criteria**:
  - Receives WS_DISCONNECTED → WS_CONNECTED events and continues sending frames

## Test Execution Order

The order of test execution should not affect results, as each test starts with a clean environment:

1. AppLaunchE2E
2. RuntimeServiceE2E
3. ProfileSwitchE2E
4. ScriptFailureRollbackE2E
5. WebSocketReconnectE2E

## Coverage Gaps

| Gap Description | Impact | Plan to Address |
|---|---|---|
| No counterexample for invalid profile ID | Medium | Add test for invalid profile ID scenario |
| No test for script timeout | Medium | Add test for script timeout scenario |
| Limited input domain coverage | Low | Add tests for different input types |

## Test Reliability

All tests must meet the following reliability criteria:
- No `Thread.sleep` (use await mechanisms instead)
- Deterministic execution (consistent results every run)
- Clean environment setup/teardown
- Strong assertions that fail clearly when expected

## Passing Threshold

For critical tests (G1, G2, G5):
- Must pass 20 consecutive runs locally
- Must pass on two different emulator configurations in CI

For non-critical tests:
- Must pass 10 consecutive runs locally
- Must pass in CI

This test matrix ensures that our E2E test suite provides strong evidence that the Input Script Runtime system functions correctly according to our defined goals.