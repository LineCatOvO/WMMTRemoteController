很好，这一步你已经不是在“选技术”，而是在**定义一套可长期演进的 UI 规范**了。  
我会直接给你一份**可以落地、可审计、可扩展、可约束社区行为的完整布局文件设计文档**，而不是零散建议。

下面这份文档假设你的前提是：

- WebView **只负责布局与渲染**
- 原生 **负责输入、命中、状态、设备**
- HTML/CSS 是布局 DSL
- JS **不是皮肤必需品**
- 所有可交互节点必须能被原生识别并命中

---

# UI 布局文件规范（WebView Skin Layout Spec）

## 1. 设计目标

### 1.1 核心目标

- 使用 **HTML/CSS** 描述 UI 布局与视觉
- 原生可 **稳定识别所有交互节点**
- 支持 **多分辨率 / 多比例**
- 支持 **社区皮肤**
- **不依赖 JS** 也能完成基础布局
- 可被 **自动审计 / 校验**

### 1.2 非目标

- 不在 HTML 中实现业务逻辑
- 不在 WebView 中处理输入
- 不在皮肤中维护状态
- 不允许皮肤访问系统能力

---

## 2. 文件结构规范

### 2.1 最小皮肤包结构（社区皮肤）

```
skin/
├── manifest.json
├── layout.html
├── style.css
└── assets/
    ├── throttle.png
    └── brake.png
```

### 2.2 官方皮肤（可选 JS）

```
skin/
├── manifest.json
├── layout.html
├── style.css
├── skin.js        # 可选，仅官方皮肤
└── assets/
```

---

## 3. manifest.json 规范

### 3.1 基本结构

```json
{
  "id": "carbon_racing_ui",
  "name": "Carbon Racing UI",
  "author": "Official",
  "version": "1.0.0",
  "type": "community",
  "supports": {
    "orientation": ["landscape"],
    "minAspectRatio": "16:9"
  }
}
```

### 3.2 type 字段

| 值 | 含义 |
|----|------|
| community | 社区皮肤（无 JS） |
| official | 官方皮肤（可含 JS） |

---

## 4. HTML 布局规范（核心）

### 4.1 基本原则

- **HTML 只描述结构**
- **CSS 决定布局与视觉**
- **所有交互节点必须显式声明**
- **禁止隐式交互**

---

## 5. UI 元素表示方式（关键）

### 5.1 通用 UI 节点规范

每一个 UI 元素必须使用以下属性：

```html
<div
  data-ui-node="throttle"
  data-ui-type="analog"
  data-ui-role="input"
></div>
```

### 5.2 必须字段说明

| 属性 | 必须 | 说明 |
|----|----|----|
| data-ui-node | 是 | 唯一 ID，用于原生命中 |
| data-ui-type | 是 | UI 类型 |
| data-ui-role | 是 | 功能角色 |

---

## 6. data-ui-type 规范（UI 类型）

### 6.1 标准类型列表

| 类型 | 说明 |
|----|----|
| analog | 连续输入（油门、方向盘） |
| button | 离散按钮 |
| toggle | 开关 |
| display | 纯显示 |
| container | 布局容器 |

### 6.2 示例

```html
<div data-ui-node="steering" data-ui-type="analog" data-ui-role="input"></div>
<div data-ui-node="gear_up" data-ui-type="button" data-ui-role="input"></div>
<div data-ui-node="speed" data-ui-type="display" data-ui-role="output"></div>
```

---

## 7. data-ui-role 规范

| role | 含义 |
|----|----|
| input | 可交互，参与命中 |
| output | 仅显示 |
| layout | 仅用于布局 |

---

## 8. 布局容器规范

### 8.1 容器声明

```html
<div data-ui-type="container" class="right-panel">
  <div data-ui-node="throttle" data-ui-type="analog" data-ui-role="input"></div>
</div>
```

### 8.2 容器不参与命中

- `container` 类型不会被原生识别为输入节点
- 仅用于 CSS 布局

---

## 9. CSS 布局规范

### 9.1 推荐布局方式

允许：

- `position: absolute`
- `flex`
- `grid`
- `aspect-ratio`
- `vw / vh`
- `min/max-width`

禁止（社区皮肤）：

- `transform: scale`
- `transform: rotate`（影响命中）
- `filter`
- `perspective`

### 9.2 示例

```css
#throttle {
  position: absolute;
  right: 5vw;
  bottom: 10vh;
  width: 12vw;
  height: 30vh;
}
```

---

## 10. 命中区域规则（非常重要）

### 10.1 命中区域定义

- 原生使用 `getBoundingClientRect()` 结果
- **不考虑 CSS transform 后的视觉变化**
- 命中区域 = 布局矩形

### 10.2 设计约束

- 所有交互元素必须是矩形
- 不允许视觉与命中区域严重不一致

---

## 11. JS 使用规范（可选）

### 11.1 JS 的职责（仅官方皮肤）

允许：

- 动画控制
- 状态展示
- 视觉效果

禁止：

- 布局计算
- 输入处理
- 网络请求
- 原生能力调用（除白名单）

### 11.2 社区皮肤

- **禁止 JS**
- WebView 使用 CSP 禁止脚本执行

---

## 12. 原生 ↔ WebView 协议（简述）

### 12.1 Web → Native

```json
{
  "type": "layoutReady",
  "nodes": [
    { "id": "throttle", "x": 120, "y": 800, "w": 200, "h": 400 }
  ]
}
```

### 12.2 Native → Web

```json
{
  "type": "stateUpdate",
  "throttle": 0.7,
  "steering": -15
}
```

---

## 13. 校验与安全规则

- HTML 必须可静态解析
- 禁止 inline JS
- 禁止外链资源
- 禁止 iframe
- 禁止 form / input
- 所有 `data-ui-node` 必须唯一

---

## 14. 设计哲学总结

> HTML 是结构  
> CSS 是布局与皮肤  
> JS 是可选增强  
> 原生是控制与真相  
> WebView 是表现层  

---

## 15. 一句话总结（给未来的你）

> **这套布局规范不是为了“用 Web 写 App”，而是为了用 Web 的成熟布局能力，构建一个安全、可控、可扩展的 UI 皮肤系统，而核心控制权始终在原生。**

如果你愿意，下一步我可以帮你：

- 写一份 **皮肤校验器规则**
- 定义 **UI 类型 → 原生输入映射表**
- 设计 **皮肤市场的审核标准**
- 或直接给你一个 **完整示例皮肤包**

你现在做的这一步，已经是平台级设计了。
