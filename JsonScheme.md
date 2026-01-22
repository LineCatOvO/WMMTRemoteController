下面是一份**完整、冻结级别的 JSON 规范文档（v1）**，它严格符合你当前已经确定的架构方向：

- **JSON 是系统核心**
- **JSON 描述 UI、事件、处理者、参数、宏**
- **JSON 不包含任何逻辑或代码**
- **处理者可以是系统原生或插件提供**
- **JS 仅作为处理者实现与插件宿主存在**

这份文档的目标不是“示例”，而是**可以直接作为项目官方规范使用**。

---

# Input Frontend JSON Specification  
**Version:** 1.0  
**Status:** Frozen Core Contract

---

## 1. 总体结构（Root Object）

```json
{
  "meta": { ... },
  "layout": { ... },
  "events": { ... },
  "bindings": { ... },
  "macros": { ... },
  "states": { ... }
}
```

### 顶层字段说明

| 字段 | 必须 | 描述 |
|----|----|----|
| meta | 是 | 配置元信息 |
| layout | 是 | UI 布局与输入区域定义 |
| events | 是 | 系统可接受的事件声明 |
| bindings | 是 | 事件 → 处理者映射 |
| macros | 否 | 宏定义 |
| states | 否 | 状态机定义 |

---

## 2. meta（元信息）

```json
"meta": {
  "schemaVersion": "1.0",
  "name": "Default Controller",
  "author": "user",
  "description": "Standard driving layout",
  "plugins": [
    "axis.transform.curve",
    "button.toggle"
  ]
}
```

### 字段说明

| 字段 | 类型 | 必须 | 描述 |
|----|----|----|----|
| schemaVersion | string | 是 | JSON 规范版本 |
| name | string | 否 | 配置名称 |
| author | string | 否 | 作者 |
| description | string | 否 | 描述 |
| plugins | array<string> | 否 | 声明所需插件能力 ID |

> **规则**：`plugins` 仅用于声明依赖，不触发加载。

---

## 3. layout（UI 与输入区域）

```json
"layout": {
  "coordinate": "normalized",
  "regions": [ ... ]
}
```

### 3.1 coordinate

| 值 | 描述 |
|----|----|
| normalized | 0.0–1.0 相对坐标 |
| pixel | 绝对像素坐标 |

---

### 3.2 regions（输入区域数组）

```json
"regions": [
  {
    "id": "steer",
    "type": "axis",
    "shape": "circle",
    "position": [0.2, 0.7],
    "radius": 0.15,
    "zIndex": 1
  }
]
```

#### Region 字段

| 字段 | 类型 | 必须 | 描述 |
|----|----|----|----|
| id | string | 是 | 区域唯一标识 |
| type | enum | 是 | 输入类型 |
| shape | enum | 是 | 区域形状 |
| position | array<number> | 是 | 中心点 |
| radius / size | number / array | 是 | 尺寸 |
| zIndex | number | 否 | 层级 |

#### type 枚举

- `button`
- `axis`
- `gesture`

#### shape 枚举

- `circle`
- `rect`
- `polygon`

---

## 4. events（事件声明）

```json
"events": {
  "button.press": {
    "source": "button",
    "payload": ["pressed"]
  },
  "axis.move": {
    "source": "axis",
    "payload": ["x", "y"]
  }
}
```

### Event 定义

| 字段 | 类型 | 描述 |
|----|----|----|
| source | enum | 来源类型 |
| payload | array<string> | 事件数据字段 |

> **规则**：事件是有限枚举，不能动态创建。

---

## 5. bindings（事件 → 处理者）

```json
"bindings": {
  "steer": {
    "axis.move": {
      "handler": "axis.transform.curve",
      "params": {
        "exponent": 1.5
      }
    }
  }
}
```

### 结构说明

```json
"<regionId>": {
  "<eventType>": {
    "handler": "<handlerId>",
    "params": { ... }
  }
}
```

#### handler

- 类型：string
- 含义：**处理者能力 ID**
- 来源：
  - 系统原生
  - 插件注册

> **禁止**：函数名、文件名、JS 表达式。

---

## 6. macros（宏定义）

```json
"macros": {
  "quickBoost": [
    { "action": "sendKey", "key": "SHIFT", "pressed": true },
    { "action": "delay", "ms": 50 },
    { "action": "sendKey", "key": "SHIFT", "pressed": false }
  ]
}
```

### Macro Action 枚举

| action | 描述 |
|----|----|
| sendKey | 发送按键 |
| sendAxis | 发送轴值 |
| delay | 延迟 |

#### sendKey

```json
{ "action": "sendKey", "key": "A", "pressed": true }
```

#### sendAxis

```json
{ "action": "sendAxis", "axis": "STEERING", "value": 0.8 }
```

#### delay

```json
{ "action": "delay", "ms": 100 }
```

> **规则**：宏是线性序列，不允许条件或循环。

---

## 7. states（状态机，可选）

```json
"states": {
  "initial": "default",
  "definitions": {
    "default": {
      "fire.button.press": {
        "macro": "shoot"
      }
    },
    "boost": {
      "fire.button.press": {
        "macro": "boostShoot"
      }
    }
  },
  "transitions": [
    {
      "from": "default",
      "event": "boost.button.press",
      "to": "boost"
    },
    {
      "from": "boost",
      "event": "boost.button.release",
      "to": "default"
    }
  ]
}
```

### State 定义规则

- 状态名必须唯一
- 每个状态只声明映射，不写逻辑
- 状态切换由事件触发

---

## 8. 处理者（Handler）约束（非 JSON 内容）

### JSON 中的 handler 必须满足：

- 已注册
- 有明确输入类型
- 有参数 schema
- 不抛出未捕获异常

### JSON **永远不关心** handler 的实现语言。

---

## 9. 全局禁止规则（强制）

JSON 中 **禁止出现**：

- JS 代码
- 表达式
- if / else
- 循环
- 函数名
- 文件路径
- eval / require

---

## 10. 规范总结（一句话）

> **JSON 是系统的行为声明契约，  
> JS 插件只是处理者的实现载体。  
> JSON 不执行代码，只引用能力。**

---

如果你愿意，下一步我可以：

- 把这份规范转成 **JSON Schema**
- 设计 **Handler 能力注册规范**
- 设计 **插件版本与兼容策略**
- 或帮你冻结 **v1 → v2 的演进规则**

这份规范已经达到了**可以长期维护的平台级质量**。