# Input Script Runtime v1.0 文档

## 系统总览

Input Script Runtime 是一个运行在 Android 平台上的输入处理框架，允许开发者通过 JavaScript 脚本定义自定义的输入映射逻辑。系统将传感器数据、触摸输入和游戏手柄输入等原始数据转换为标准化的输入帧，通过 WebSocket 发送到后端。

## 核心概念

### Runtime

Runtime 是整个系统的核心，负责管理输入采集、脚本执行和 WebSocket 通信。它以 Service 形式运行，独立于 UI 生命周期，确保即使在 Activity 被销毁时也能持续处理输入。

### Profile

Profile 是一组配置和脚本的集合，定义了特定的输入映射规则。每个 Profile 包含：
- 元数据配置文件（profile.json）
- JavaScript 脚本文件（main.js）

### Script Engine

Script Engine 负责执行 Profile 中的 JavaScript 脚本，将原始输入转换为标准化的输入帧。当前实现基于 WebView，提供安全的沙箱环境。

### WebSocket Protocol

系统通过 WebSocket 将处理后的输入帧发送到后端，采用 AsyncAPI 定义的消息格式，确保跨平台兼容性和可扩展性。

## 文档导航

### 协议规范

- [AsyncAPI 定义](asyncapi.yaml)：WebSocket 协议的机器可读定义

### 运行时行为

- [Runtime 行为规范](runtime.md)：解释 Runtime 的核心行为和执行模型
- [Profile 规范](profiles.md)：Profile 目录结构和脚本编写指南
- [错误模型与回退策略](errors.md)：系统的错误处理机制
- [生命周期管理](lifecycle.md)：系统各组件的生命周期流程
- [版本与兼容策略](versioning.md)：版本号格式和兼容性保证

## 快速入门

1. **了解核心概念**：阅读本页的核心概念介绍
2. **熟悉协议格式**：查看 [AsyncAPI 定义](asyncapi.yaml) 了解消息结构
3. **理解运行时行为**：阅读 [Runtime 行为规范](runtime.md)
4. **学习 Profile 开发**：查看 [Profile 规范](profiles.md) 学习如何编写自定义 Profile
5. **掌握错误处理**：了解系统的 [错误模型与回退策略](errors.md)

## 版本信息

当前版本：v1.0.0

发布日期：2026-01-21

## 开发团队

LineCat

## 许可证

MIT License
