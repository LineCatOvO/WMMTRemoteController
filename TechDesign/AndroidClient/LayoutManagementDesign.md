# Android客户端布局管理技术设计文档

## 概述

本文档详细说明Android客户端布局管理系统的设计与实现方案，重点描述布局从外部存储文件夹读取的机制。

## 设计目标

1. 实现从应用外部存储文件夹动态加载布局
2. 支持多布局管理，一个JSON文件对应一个布局
3. JSON的metadata部分描述布局元数据（名称、描述等）
4. 布局管理页面从外部存储文件夹读取所有JSON文件，验证合法性后显示给用户
5. 新用户可从预定义布局中选择一个作为初始布局
6. 预定义布局存放在 `predefined-profiles` 目录
7. 实现UI组件与系统节点的分离，确保架构清晰

## 系统架构

### 1. 布局存储结构

```
外部存储/Android/data/包名/files/layout/
├── layout1/
│   └── main.json
├── layout2/
│   └── main.json
└── ...
```

### 2. 布局JSON格式

```json
{
  "layoutId": "unique_layout_identifier",
  "version": 1,
  "description": "布局描述信息",
  "metadata": {
    "name": "布局名称",
    "author": "作者",
    "version": "1.0.0",
    "description": "详细描述"
  },
  "elements": [
    {
      "id": "element_id",
      "type": "button|analog|gyro",
      "position": {
        "x": 0.5,
        "y": 0.5
      },
      "size": {
        "width": 0.2,
        "height": 0.1
      },
      "mapping": {
        "button": "A|B|X|Y|LB|RB|LT|RT|DPAD_UP|DPAD_DOWN|DPAD_LEFT|DPAD_RIGHT|START|BACK|LS|RS",
        "axis": "LX|LY|RX|RY",
        "trigger": "LT|RT"
      }
    }
  ]
}
```

### 3. 预定义布局结构

```
assets/predefined-profiles/
├── basic_racing_layout/
│   └── main.json
├── drift_control_layout/
│   └── main.json
└── ...
```

## 核心组件

### 1. LayoutManager

负责管理外部存储中的布局项目，提供以下功能：

- 读取外部存储中的所有布局项目
- 验证布局JSON格式的合法性
- 创建、更新、删除布局项目
- 提供布局项目的元数据信息

### 2. LayoutLoader

负责从外部存储加载布局文件：

- 扫描外部存储布局目录
- 验证JSON格式
- 解析布局数据
- 缓存已加载的布局

### 3. LayoutValidator

负责验证布局文件的合法性：

- 验证JSON格式
- 检查必要字段是否存在
- 验证元素配置的有效性
- 检查坐标和尺寸的合理性

### 4. LayoutRenderer (UI层)

负责将布局渲染到屏幕上并处理触摸事件：

- **职责**：呈现布局元素，处理用户交互
- **不负责**：生成操作意图、访问设备映射、处理原始输入
- 仅将交互事件报告给交互捕获层

### 5. 交互捕获层 (InteractionCapture)

- **职责**：捕获和标准化原始输入
- **功能**：处理来自UI层的交互事件，转换为标准化输入
- **特点**：与UI解耦，专注于输入处理

### 6. 抽象操作层 (IntentComposer)

- **职责**：将原始输入转换为抽象操作意图
- **功能**：应用死区、平滑、曲线等处理算法
- **特点**：纯逻辑处理，与UI无关

### 7. 设备映射层 (DeviceProjector)

- **职责**：将抽象操作意图投影到具体设备输出
- **功能**：与服务端通信，发送输入状态
- **特点**：确定性输出，与UI无关

## 实现细节

### 1. 布局加载流程

1. 应用启动时，LayoutManager扫描外部存储的layout目录
2. 遍历每个子目录，读取main.json文件
3. 使用LayoutValidator验证JSON格式和内容
4. 解析布局元数据并构建布局列表
5. 在布局管理页面显示可用布局

### 2. 预定义布局处理

1. 新用户首次启动应用时，外部存储中无自定义布局
2. 应用从assets/predefined-profiles目录复制一份预定义布局到外部存储
3. 用户可在创建页面选择预定义布局

### 3. 布局验证机制

- JSON格式验证
- 必需字段检查（layoutId, version, elements等）
- 坐标范围验证（0.0-1.0）
- 尺寸有效性检查
- 元素类型验证

### 4. UI 与系统节点分离实现

#### 4.1 UI 组件职责

UI 组件（如布局渲染器）仅负责：

- 呈现布局元素
- 命中测试
- 将交互事件报告给交互捕获层

UI 组件不应：

- 直接生成 Operation
- 直接访问 LayoutEngine
- 直接绑定设备语义
- 直接处理原始输入源

#### 4.2 系统节点职责

系统节点（如 Interaction/Operation Node）负责：

- 由布局 JSON 定义
- 与 UI 解耦
- 可测试、可序列化、可回放

系统节点处理：

- 接收来自 InputSource 的标准化事件
- 根据布局配置决定是否命中和如何转换
- 输出 Operation

## 用户体验

### 1. 首次使用流程

1. 新用户打开应用时，系统检测外部存储中是否有布局
2. 如无布局，提示用户从预定义布局中选择
3. 选择后复制到外部存储并激活使用

### 2. 布局管理页面

1. 显示所有可用布局（来自外部存储）
2. 显示布局名称、描述等元数据
3. 支持选择、预览、删除布局
4. 提供创建新布局的功能

## 数据持久化

- 布局文件存储在应用外部存储目录
- 路径：`Context.getExternalFilesDir(null)/"layout"/{layoutId}/main.json`
- 保证应用卸载时数据被清除
- 支持用户备份和迁移布局文件

## 安全考虑

- 验证所有从外部存储读取的布局文件
- 防止恶意JSON导致的安全漏洞
- 限制布局元素的坐标和尺寸范围
- 验证映射配置的有效性

## 错误处理

- 布局文件损坏时使用默认布局
- 验证失败时提示用户并跳过无效布局
- 外部存储不可用时使用内置备用布局

## 扩展性

- 支持未来新增布局元素类型
- 支持布局版本升级机制
- 支持布局导入导出功能
- 支持UI与逻辑层的独立扩展
