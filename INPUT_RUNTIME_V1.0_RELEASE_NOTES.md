# Input Script Runtime v1.0 发布说明

## 1. 封板声明

> **Input Script Runtime v1.0 已完成并正式封板。**

自发布之日起，v1.0版本的以下内容将永久冻结：

- **ABI**：应用二进制接口不再修改
- **Stable API**：稳定API不再增加新功能，仅进行bug修复
- **Profile 语义**：配置文件格式和生命周期不再修改
- **Runtime 行为保证**：核心运行时行为不再变更

## 2. 已完成功能

### 2.1 核心架构
- ✅ 基于WebView的JavaScript脚本引擎
- ✅ 60 FPS的稳定帧序列执行
- ✅ 确定性执行模型
- ✅ 前台服务化运行时

### 2.2 API设计
- ✅ ScriptContext拆权设计
  - **RawAccess**：只读访问接口（Stable）
  - **StateMutator**：状态修改接口（Stable）
  - **HostServices**：主机服务接口（Experimental）
- ✅ @Stable/@Experimental API分类
- ✅ 明确的API版本控制

### 2.3 Profile管理
- ✅ Profile加载与切换
- ✅ 原子切换语义
- ✅ 自动回滚机制
- ✅ 完整的错误处理
- ✅ Profile导入导出功能

### 2.4 测试与验证
- ✅ ExtremeCaseTests极端情况测试
- ✅ ProfileManagerContractTests契约测试
- ✅ ScriptTestHarness测试框架
- ✅ 粘键守护测试
- ✅ 结构化TestResult报告

### 2.5 官方参考Profile
- ✅ **wmmt_keyboard_basic**：基础键盘输入配置
- ✅ **wmmt_gamepad_standard**：标准游戏手柄配置
- ✅ **wmmt_gyro_touch_combo**：陀螺仪+触摸组合配置

## 3. Stable API列表

### 3.1 RawAccess接口
```java
@Stable
public interface RawAccess {
    RawInput getRawInput();
    float getAxis(String axisName);
    boolean isGamepadButtonPressed(String buttonName);
    long getFrameId();
    long getTimestamp();
}
```

### 3.2 StateMutator接口
```java
@Stable
public interface StateMutator {
    void holdKey(String key);
    void releaseKey(String key);
    void releaseAllKeys();
    boolean isKeyHeld(String key);
    void pushEvent(String eventType, Object eventData);
}
```

### 3.3 HostServices接口
```java
@Experimental
public interface HostServices {
    void log(String message);
    void debug(String message);
    void error(String message, Throwable error);
    void setMousePosition(float x, float y);
    void setMouseButton(String button, boolean pressed);
    ScriptContext.GyroData getGyro();
    ScriptContext.TouchData getTouch();
    Object getProfileMetadata(String key);
    boolean requestPermission(String permission);
}
```

## 4. Profile格式规范

### 4.1 核心字段
- `name`：配置文件名称
- `version`：版本号
- `author`：作者
- `engineApiVersion`：引擎API版本
- `entry`：脚本入口点
- `scriptCode`：脚本代码

### 4.2 生命周期
- **加载**：通过ProfileManager.loadProfile()加载
- **切换**：通过ProfileManager.switchProfile()原子切换
- **更新**：每帧调用脚本update()函数
- **重置**：脚本错误或切换时调用reset()

## 5. 后续维护策略

### 5.1 v1.0分支
- 仅接受bug修复
- 不接受新功能添加
- 所有修改需经过完整测试

### 5.2 未来版本计划
- v1.x：bug修复和性能优化
- v2.0：新功能开发（需明确的版本升级策略）

## 6. 官方参考Profile使用指南

### 6.1 目录结构
```
official-profiles/
├── wmmt_keyboard_basic/
│   ├── profile.json
│   └── main.js
├── wmmt_gamepad_standard/
│   ├── profile.json
│   └── main.js
└── wmmt_gyro_touch_combo/
    ├── profile.json
    └── main.js
```

### 6.2 使用方法
1. 将官方参考Profile复制到设备存储
2. 通过ProfileManager.importProfileFromZip()导入
3. 使用ProfileManager.switchProfile()切换到目标Profile

## 7. 迁移指南

### 7.1 从旧版本迁移
- 检查engineApiVersion是否兼容
- 确保Profile格式符合v1.0规范
- 更新脚本以使用Stable API

## 8. 联系方式

如有问题或建议，请通过以下方式联系：
- 项目仓库：[GitHub Repository]
- 邮件：[Contact Email]

## 9. 许可证

Input Script Runtime v1.0采用[许可证类型]许可证。

---

**发布日期**：2026-01-21  
**发布版本**：v1.0.0  
**发布状态**：正式封板  
**维护状态**：稳定维护中