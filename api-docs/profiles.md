# Profile 规范

## 1. Profile 概述

Profile 是 Input Script Runtime 系统中的核心概念，它定义了如何将原始输入数据映射为标准化的 InputFrame。每个 Profile 包含配置文件和 JavaScript 脚本，用于实现特定的输入映射逻辑。

## 2. Profile 目录结构

每个 Profile 必须遵循以下目录结构：

```text
/profile_name
├─ profile.json    # 元数据配置文件
└─ main.js         # JavaScript 脚本文件
```

### 2.1 Profile 命名规则

- 名称必须唯一，不允许重复
- 只能包含字母、数字、下划线和连字符
- 长度限制：1-64 个字符
- 建议使用描述性名称，如 `racing_game_standard` 或 `fps_aim_assist`

## 3. profile.json 规范

`profile.json` 是 Profile 的元数据配置文件，包含以下字段：

### 3.1 必需字段

| 字段名 | 类型 | 描述 | 示例 |
|--------|------|------|------|
| `name` | string | Profile 名称 | "racing_standard" |
| `version` | string | Profile 版本号 | "1.0.0" |
| `description` | string | Profile 描述 | "Standard racing game profile" |
| `author` | string | Profile 作者 | "LineCat" |
| `apiVersion` | string | 兼容的 API 版本 | "1.0" |

### 3.2 可选字段

| 字段名 | 类型 | 描述 | 示例 |
|--------|------|------|------|
| `default` | boolean | 是否为默认 Profile | false |
| `category` | string | Profile 分类 | "racing" |
| `tags` | array | 标签列表 | ["racing", "gamepad"] |
| `compatibility` | object | 兼容性信息 | {"minRuntimeVersion": "1.0.0"} |
| `settings` | object | 可配置的设置项 | {"sensitivity": 1.0} |

### 3.3 示例 profile.json

```json
{
  "name": "racing_standard",
  "version": "1.0.0",
  "description": "Standard input profile for racing games",
  "author": "LineCat",
  "apiVersion": "1.0",
  "default": false,
  "category": "racing",
  "tags": ["racing", "gamepad", "gyro"],
  "compatibility": {
    "minRuntimeVersion": "1.0.0"
  },
  "settings": {
    "steeringSensitivity": 1.0,
    "throttleSensitivity": 1.0,
    "useGyro": true
  }
}
```

## 4. main.js 脚本规范

`main.js` 是 Profile 的核心脚本文件，负责实现输入映射逻辑。脚本运行在 WebView 沙箱环境中，只能访问 Runtime 提供的有限 API。

### 4.1 脚本结构

脚本必须导出一个 `Profile` 对象，包含以下方法：

```javascript
const Profile = {
  // 初始化函数，在 Profile 加载时调用
  init(config) {
    // 初始化逻辑
  },
  
  // 主输入处理函数，每次输入更新时调用
  update(rawInput) {
    // 输入映射逻辑
    return {
      keyboard: [],
      mouse: { x: 0, y: 0, left: false, right: false, middle: false },
      joystick: { x: 0, y: 0 },
      gyroscope: { pitch: 0, roll: 0, yaw: 0 }
    };
  },
  
  // 清理函数，在 Profile 卸载时调用
  cleanup() {
    // 清理逻辑
  }
};
```

### 4.2 初始化函数 (init)

- **参数**：`config` - Profile 配置对象，包含 `profile.json` 中的 `settings` 字段
- **返回值**：无
- **调用时机**：Profile 加载时
- **用途**：初始化脚本状态、设置默认值、配置参数

### 4.3 主输入处理函数 (update)

- **参数**：`rawInput` - 预处理后的原始输入对象
- **返回值**：标准化的 InputFrame 对象
- **调用时机**：每次输入更新时（约 60 次/秒）
- **用途**：将原始输入映射为标准化的输入帧

### 4.4 清理函数 (cleanup)

- **参数**：无
- **返回值**：无
- **调用时机**：Profile 卸载时
- **用途**：清理资源、保存状态

## 5. 输入数据格式

### 5.1 rawInput 对象结构

`rawInput` 对象包含以下字段：

```javascript
{
  // 传感器数据
  sensors: {
    // 加速度计数据 (m/s²)
    accelerometer: {
      x: number,
      y: number,
      z: number
    },
    // 陀螺仪数据 (rad/s)
    gyroscope: {
      x: number,
      y: number,
      z: number
    },
    // 方向传感器数据 (degrees)
    orientation: {
      azimuth: number,
      pitch: number,
      roll: number
    }
  },
  
  // 触摸输入数据
  touch: {
    // 触摸点列表
    points: [
      {
        id: number,
        x: number,  // 归一化坐标 (0-1)
        y: number,  // 归一化坐标 (0-1)
        pressure: number,  // 压力值 (0-1)
        state: string  // "down", "move", "up"
      }
    ],
    // 触摸事件类型
    eventType: string  // "down", "move", "up", "cancel"
  },
  
  // 游戏手柄数据
  gamepad: {
    // 按键状态
    buttons: {
      a: boolean,
      b: boolean,
      x: boolean,
      y: boolean,
      lb: boolean,
      rb: boolean,
      lt: number,  // 扳机键值 (0-1)
      rt: number,  // 扳机键值 (0-1)
      back: boolean,
      start: boolean,
      leftStick: boolean,
      rightStick: boolean,
      dpadUp: boolean,
      dpadDown: boolean,
      dpadLeft: boolean,
      dpadRight: boolean
    },
    // 摇杆数据
    sticks: {
      left: {
        x: number,  // 归一化值 (-1 到 1)
        y: number   // 归一化值 (-1 到 1)
      },
      right: {
        x: number,
        y: number
      }
    }
  },
  
  // 设备状态
  device: {
    // 设备类型
    type: string,  // "phone", "tablet", "tv"
    // 屏幕方向
    orientation: string,  // "portrait", "landscape"
    // 电池状态
    battery: {
      level: number,  // 电池电量 (0-1)
      charging: boolean
    }
  }
}
```

### 5.2 返回值格式

`update` 函数必须返回以下格式的对象：

```javascript
{
  // 键盘状态：当前按下的按键列表
  keyboard: [string],
  
  // 鼠标状态
  mouse: {
    x: number,  // 归一化坐标 (0-1)
    y: number,  // 归一化坐标 (0-1)
    left: boolean,  // 左键状态
    right: boolean,  // 右键状态
    middle: boolean  // 中键状态
  },
  
  // 摇杆状态
  joystick: {
    x: number,  // 归一化值 (-1 到 1)
    y: number   // 归一化值 (-1 到 1)
  },
  
  // 陀螺仪状态
  gyroscope: {
    pitch: number,  // 俯仰角 (degrees)
    roll: number,   // 横滚角 (degrees)
    yaw: number     // 偏航角 (degrees)
  }
}
```

## 6. 可用 API

脚本可以使用以下 Runtime 提供的 API：

### 6.1 日志 API

```javascript
// 输出日志信息
log(message);

// 输出调试信息
debug(message);

// 输出警告信息
warn(message);

// 输出错误信息
error(message);
```

### 6.2 状态管理 API

```javascript
// 获取当前输入状态
const currentState = getState();

// 设置输入状态
setState(newState);

// 重置输入状态
resetState();
```

### 6.3 配置 API

```javascript
// 获取当前配置
const config = getConfig();

// 更新配置
updateConfig(newConfig);
```

### 6.4 工具函数 API

```javascript
// 线性映射
const mappedValue = map(value, inMin, inMax, outMin, outMax);

// 限制值范围
const clampedValue = clamp(value, min, max);

// 平滑过滤
const smoothedValue = smooth(current, previous, factor);

// 计算距离
const distance = distance(x1, y1, x2, y2);
```

## 7. 脚本编写最佳实践

### 7.1 性能优化

- **避免复杂计算**：单次脚本执行时间不应超过 10ms
- **缓存计算结果**：对于不变的值，避免重复计算
- **减少对象创建**：避免在 update 函数中频繁创建新对象
- **使用高效算法**：选择时间复杂度较低的算法

### 7.2 代码结构

- **模块化设计**：将复杂逻辑拆分为多个函数
- **清晰的命名**：使用描述性的变量和函数名
- **注释**：添加必要的注释，解释复杂逻辑
- **错误处理**：使用 try-catch 块处理可能的异常

### 7.3 输入处理

- **平滑输入**：对传感器数据进行滤波，减少抖动
- **归一化处理**：确保输出值在规定范围内
- **考虑边缘情况**：处理极端输入值
- **提供默认值**：确保所有输出字段都有合理的默认值

### 7.4 兼容性

- **版本检查**：检查 API 版本，确保兼容性
- **优雅降级**：当遇到不支持的特性时，优雅降级
- **向后兼容**：尽量保持与旧版本的兼容性

## 8. 示例 Profile

### 8.1 racing_standard Profile

#### profile.json

```json
{
  "name": "racing_standard",
  "version": "1.0.0",
  "description": "Standard racing game profile with gyro steering",
  "author": "LineCat",
  "apiVersion": "1.0",
  "category": "racing",
  "tags": ["racing", "gyro", "touch"],
  "settings": {
    "steeringSensitivity": 1.0,
    "throttleSensitivity": 1.0,
    "brakeSensitivity": 1.0,
    "useGyro": true
  }
}
```

#### main.js

```javascript
const Profile = {
  // 保存上一次的状态
  lastState: {
    keyboard: [],
    mouse: { x: 0.5, y: 0.5, left: false, right: false, middle: false },
    joystick: { x: 0, y: 0 },
    gyroscope: { pitch: 0, roll: 0, yaw: 0 }
  },
  
  // 保存配置
  config: {},
  
  init(config) {
    this.config = config;
    log(`Initialized racing_standard profile with config: ${JSON.stringify(config)}`);
  },
  
  update(rawInput) {
    const state = {
      keyboard: [],
      mouse: { ...this.lastState.mouse },
      joystick: { x: 0, y: 0 },
      gyroscope: { ...rawInput.sensors.orientation }
    };
    
    // 处理陀螺仪转向
    if (this.config.useGyro) {
      // 将陀螺仪横滚角映射为转向值 (-1 到 1)
      state.joystick.x = map(rawInput.sensors.orientation.roll, -45, 45, -1, 1);
    }
    
    // 处理触摸输入（油门和刹车）
    if (rawInput.touch.points.length > 0) {
      const touch = rawInput.touch.points[0];
      
      // 屏幕左侧：刹车
      if (touch.x < 0.3) {
        state.joystick.y = map(touch.y, 0, 1, 1, 0); // 从底部到顶部：刹车强度 1 到 0
        state.keyboard.push("S");
      }
      // 屏幕右侧：油门
      else if (touch.x > 0.7) {
        state.joystick.y = map(touch.y, 0, 1, -1, 0); // 从底部到顶部：油门强度 1 到 0
        state.keyboard.push("W");
      }
    }
    
    // 处理游戏手柄（如果可用）
    if (rawInput.gamepad) {
      // 右摇杆：转向
      state.joystick.x = rawInput.gamepad.sticks.right.x;
      // 右扳机：油门
      if (rawInput.gamepad.buttons.rt > 0.1) {
        state.joystick.y = -rawInput.gamepad.buttons.rt;
        state.keyboard.push("W");
      }
      // 左扳机：刹车
      if (rawInput.gamepad.buttons.lt > 0.1) {
        state.joystick.y = rawInput.gamepad.buttons.lt;
        state.keyboard.push("S");
      }
      // A 按钮：加速
      if (rawInput.gamepad.buttons.a) {
        state.keyboard.push("W");
      }
      // B 按钮：刹车
      if (rawInput.gamepad.buttons.b) {
        state.keyboard.push("S");
      }
    }
    
    // 限制摇杆值范围
    state.joystick.x = clamp(state.joystick.x, -1, 1);
    state.joystick.y = clamp(state.joystick.y, -1, 1);
    
    // 保存当前状态
    this.lastState = state;
    
    return state;
  },
  
  cleanup() {
    log("Cleaned up racing_standard profile");
  }
};
```

## 9. Profile 验证规则

在加载 Profile 时，系统会执行以下验证：

1. **目录结构验证**：检查是否包含必需的文件
2. **profile.json 验证**：
   - 检查必需字段是否存在
   - 验证字段类型和格式
   - 检查版本号格式
3. **main.js 验证**：
   - 检查是否导出了 Profile 对象
   - 验证必需的方法是否存在
   - 检查语法错误
   - 执行简单的功能测试
4. **API 兼容性验证**：检查是否使用了兼容的 API 版本

## 10. Profile 生命周期

### 10.1 加载过程

1. **发现 Profile**：系统扫描 Profile 目录，发现所有可用的 Profile
2. **验证 Profile**：对每个 Profile 执行验证
3. **选择默认 Profile**：根据配置选择默认 Profile
4. **初始化 Profile**：调用 init() 方法
5. **激活 Profile**：开始执行 update() 方法

### 10.2 切换过程

1. **清理当前 Profile**：调用当前 Profile 的 cleanup() 方法
2. **加载新 Profile**：验证并初始化新 Profile
3. **激活新 Profile**：开始执行新 Profile 的 update() 方法

### 10.3 卸载过程

1. **停止执行**：停止调用 update() 方法
2. **清理资源**：调用 cleanup() 方法
3. **释放内存**：释放 Profile 占用的内存

## 11. Profile 管理

### 11.1 安装 Profile

- **手动安装**：将 Profile 目录复制到系统的 Profile 目录
- **自动安装**：通过应用内下载或导入功能安装
- **验证安装**：系统会自动验证安装的 Profile

### 11.2 更新 Profile

- **手动更新**：替换 Profile 目录中的文件
- **自动更新**：通过应用内更新功能更新
- **版本管理**：系统保留旧版本，支持回滚

### 11.3 删除 Profile

- **手动删除**：删除 Profile 目录
- **应用内删除**：通过应用内管理界面删除
- **默认 Profile 保护**：系统默认 Profile 无法删除

## 12. 安全考虑

- **沙箱环境**：脚本在隔离的 WebView 环境中执行，无法访问设备资源
- **API 限制**：仅暴露必要的 API，限制潜在风险
- **输入验证**：严格验证脚本返回的 InputFrame 数据格式
- **执行时间限制**：单次脚本执行时间超过 100ms 时会被强制终止
- **内存限制**：单个 Profile 占用的内存不超过 64MB
