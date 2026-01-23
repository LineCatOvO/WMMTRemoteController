# UI 与系统节点分离设计规范

## 1. 设计原则

### 1.1 核心理念
>
> **UI 组件 ≠ 系统节点**

UI 组件是**表现层对象**，而系统节点（Node）是逻辑实体，参与状态生成、测试、回放和组合。为避免系统在演进过程中变成"UI + 输入 + 逻辑 + 状态"纠缠在一起的状态，必须严格遵守以下设计原则：

1. **UI 组件只负责**：
   - 呈现
   - 命中测试
   - 把"发生了什么交互"报告出去

2. **UI 组件不应该**：
   - 直接生成 Operation
   - 直接访问 LayoutEngine
   - 直接绑定设备语义
   - 直接处理原始输入源

### 1.2 架构判断标准

> **如果我把 UI 全部换成另一套（甚至无 UI），  
> LayoutEngine 和 Operation 是否还能工作？**

- 如果答案是 **能** → 架构是对的
- 如果答案是 **不能** → UI 已经越权了

## 2. 系统架构分层

### 2.1 原始输入模块（全局、少量、稳定）

```
TouchInputSource
GyroInputSource
KeyInputSource
```

- 每种输入源一个模块
- 负责：采集、标准化、时间戳
- **完全不知道布局和 UI**

### 2.2 UI 组件作为"Interaction Surface"

UI 组件应该是**被动的**：

```
ThrottleView
BrakeView
GearButtonView
SteeringWheelView
```

它们只做一件事：

> **声明"我对哪些 Interaction 感兴趣"**

例如：

- 一个区域
- 一个手势
- 一个方向
- 一个可视反馈

### 2.3 系统节点（Interaction/Operation Node）

**Node 存在于逻辑层，而非 UI 层**，它们是：

- 由布局 JSON 定义
- 与 UI 解耦
- 可测试、可序列化、可回放

例如：

```
AnalogControlNode
ButtonControlNode
GyroControlNode
```

它们：

- 接收来自 InputSource 的标准化事件
- 根据布局配置决定：
  - 是否命中
  - 如何转换
- 输出 Operation

## 3. 推荐的系统流

```
少量 InputSource → 多个 Interaction/Operation Node → IntentComposer → DeviceProjector
```

UI 只是这些 Node 的"皮肤"，不是它们的灵魂。

## 4. 代码实现指南

### 4.1 UI 组件实现

UI 组件（如 LayoutRenderer）应该：

1. 只处理视觉呈现
2. 将交互事件转发给适当的系统组件
3. 不直接操作输入状态
4. 不直接访问设备输出

示例：

```java
public class LayoutRenderer extends View {
    // 只处理绘制和触摸事件
    // 将交互事件发送给 InteractionCapture
    private void sendButtonEventToController(String buttonId, boolean isPressed) {
        if (inputController != null) {
            inputController.triggerProjection();
        }
    }
}
```

### 4.2 系统节点实现

系统节点（如 InteractionCapture）应该：

1. 处理来自UI的标准化事件
2. 维护输入状态
3. 协调 IntentComposer 和 DeviceProjector
4. 与UI完全解耦

示例：

```java
public class InteractionCapture {
    // 处理原始输入事件
    // 协调意图合成和设备投影
    public void triggerProjection() {
        if (isInitialized && intentComposer != null && deviceProjector != null) {
            // 1. 使用意图合成器处理输入
            RawInput processedInput = intentComposer.composeIntent(collect());
            // 2. 使用设备投影器将处理后的输入投影到设备
            deviceProjector.projectToDevice(processedInput, frameIdCounter++);
        }
    }
}
```

## 5. 布局文件设计

布局文件（JSON）应该定义逻辑节点，而不是UI组件：

```json
{
  "meta": {},
  "ui": {
    "throttle_region": {
      "type": "button",
      "area": {"x": 0.8, "y": 0.5, "width": 0.15, "height": 0.3},
      "zIndex": 1
    }
  },
  "operations": {
    "throttle_operation": {
      "type": "axis",
      "range": {"min": 0.0, "max": 1.0},
      "default": 0.0
    }
  },
  "mappings": {
    "throttle_to_key": {
      "operation": "throttle_operation",
      "target": {
        "device": "keyboard",
        "key": "W"
      }
    }
  }
}
```

## 6. 测试策略

### 6.1 UI 组件测试

- 测试视觉呈现是否正确
- 测试交互事件是否正确转发
- 不测试输入处理逻辑

### 6.2 系统节点测试

- 测试输入处理逻辑
- 测试状态转换
- 与UI组件解耦测试

### 6.3 集成测试

- 测试UI与系统节点的交互
- 验证整体功能流程

## 7. 常见反模式

### 7.1 UI 组件承担过多职责

- ❌ UI 组件直接操作设备输出
- ❌ UI 组件直接访问布局引擎
- ❌ UI 组件维护复杂状态

### 7.2 系统与表现混淆

- ❌ 布局文件定义UI样式属性
- ❌ 输入处理逻辑嵌入UI组件
- ❌ 设备映射与UI紧密耦合

## 8. 总结

UI 与系统节点分离是确保系统长期可维护性的关键设计决策。通过保持这种分离，我们可以：

- 独立开发和测试UI与逻辑
- 灵活更换UI而不影响系统功能
- 保持清晰的架构边界
- 降低系统复杂度
- 提高代码可测试性
