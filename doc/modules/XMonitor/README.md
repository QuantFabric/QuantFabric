# XMonitor 文档索引

## 📚 文档列表

### 1. [GUI 架构与插件系统](01_GUI_Architecture.md)
深入理解 XMonitor 插件化设计

**内容**:
- 设计理念（模块化、可定制、高性能、易扩展）
- 插件架构图
- 核心组件
  - 插件接口定义（IPlugin）
  - 插件管理器（PluginManager）
  - 可拖拽布局（QDockWidget）
- 插件实现示例
  - Market 插件（行情展示）
  - OrderManager 插件（交易管理）
- 性能优化
  - 虚拟列表（只渲染可见行）
  - 批量更新（减少信号触发）
  - 异步加载（后台线程）

**适合**:
- Qt 开发者
- GUI 架构师
- 插件开发者

### 2. [基础文档 - 模块概览](../XMonitor.md)
快速了解 XMonitor

**内容**:
- 模块概述
- 主要插件介绍
- 编译运行

**适合**:
- 初学者
- 快速参考

## 🎯 核心插件

| 插件 | 功能 | 技术要点 |
|------|------|---------|
| **Permission** | 权限管理 | 登录认证、插件权限配置 |
| **Market** | 行情监控 | 虚拟列表、实时更新 |
| **EventLog** | 事件日志 | 日志过滤、搜索 |
| **Monitor** | 性能监控 | 图表展示、进程管理 |
| **RiskJudge** | 风控管理 | 参数设置、统计展示 |
| **OrderManager** | 订单管理 | 手动报单、订单查询 |
| **FutureAnalysis** | 期货分析 | 仓位汇总、盈亏统计 |
| **StockAnalysis** | 股票分析 | 持仓分析、T+0 统计 |

## 📊 架构优势

| 特性 | 说明 |
|------|------|
| 插件化 | 功能模块化，易于扩展 |
| 可定制 | 用户自定义布局和显示 |
| 高性能 | 虚拟渲染，支持海量数据 |
| 跨平台 | Windows/Linux 通用 |

## 🛠️ 开发指南

### 开发新插件

1. 继承 `IPlugin` 接口
2. 实现必需方法（GetName/GetWidget/OnMessage）
3. 创建 UI 界面（QWidget/QTableView/等）
4. 处理消息（OnMessage）
5. 注册到 PluginManager

### 自定义界面

1. 使用 QDockWidget 实现可拖拽布局
2. 使用 QSettings 保存/加载布局
3. 自定义 QTableView/QAbstractTableModel

## 📖 相关文档

- [XServer 中间件架构](../XServer/01_Middleware%20Architecture.md) - 服务端实现
- [FinTechUI 组件](https://github.com/QuantFabric/FinTechUI) - UI 组件库

## 🔗 技术栈

- GUI 框架: Qt 5.12+
- 布局系统: QDockWidget
- 数据模型: QAbstractTableModel
- 网络通信: HPSocket
- 信号槽: Qt Signal/Slot
