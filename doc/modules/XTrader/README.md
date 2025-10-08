# XTrader 文档索引

## 📚 文档列表

### 1. [设计架构与原理](01_DesignArchitecture.md)
深入理解 XTrader 交易执行引擎

**内容**:
- 设计理念（极致低延迟、高可靠性、多柜台支持）
- 架构设计图
- 架构分层（应用层、插件层、通信层）
- 数据流向设计（策略报单、手动报单、撤单流程）
- 线程模型（WorkThread、MonitorThread）
- 订单状态管理
- 性能优化策略
- 可靠性保证

**适合**:
- 系统架构师
- 交易系统开发者
- 性能优化工程师

### 2. [技术实现详解](02_TechnicalImplementation.md)
核心技术的具体实现

**内容**:
- 插件加载机制（dlopen/dlsym、符号导出）
- 共享内存通信（OrderServer、SPSC 队列）
- 风控交互机制（RiskServer 连接、检查流程）
- 网络通信实现（HPPackClient、Pack 协议）
- 订单管理实现（OrderManager、仓位、资金）
- CTP 插件实现（完整示例）
- 性能优化技术（CPU 缓存、编译器优化、系统调用）

**适合**:
- C++ 开发者
- 插件开发者
- 想学习交易系统实现的读者

### 3. [插件开发指南](03_PluginDevelopment.md)
手把手教你开发交易插件

**内容**:
- 完整开发流程（6 个步骤）
- 插件接口实现（TradeGateWay）
- 柜台 API 适配（数据转换）
- CMake 编译配置
- 调试技巧（GDB、日志、内存检查）
- 最佳实践（错误处理、线程安全、性能优化）
- 常见问题排查

**适合**:
- 插件开发者
- 需要对接新柜台的工程师
- 想扩展系统功能的开发者

### 4. [基础文档 - 模块概览](../XTrader.md)
快速了解 XTrader

**内容**:
- 模块概述
- 核心功能
- 插件架构
- 配置文件
- 编译运行

**适合**:
- 初学者
- 快速查阅参考

## 🎯 学习路径

### 新手入门
1. 先阅读 [模块概览](../XTrader.md) 了解基本概念
2. 查看配置文件和运行方式
3. 运行 Demo 体验功能

### 深入学习
1. 阅读 [设计架构与原理](01_DesignArchitecture.md) 理解设计思想
2. 学习 [技术实现详解](02_TechnicalImplementation.md) 掌握核心技术
3. 结合源码深入分析

### 插件开发
1. 阅读 [插件开发指南](03_PluginDevelopment.md)
2. 参考 CTP 插件实现
3. 开发自己的柜台插件

## 🎯 核心功能

### 交易执行
- **接收报单**: 从 XQuant (共享内存) 和 XMonitor (网络)
- **风控检查**: 与 XRiskJudge 实时交互
- **柜台执行**: 调用 CTP/REM/YD/OES API
- **回报推送**: 双路径推送 (共享内存 + 网络)

### 订单管理
- **状态追踪**: 精确订单状态机
- **仓位管理**: 实时更新多空仓位
- **资金管理**: 追踪可用资金、保证金、盈亏

### 多柜台支持
- **期货**: CTP, REM, YD
- **股票**: OES, XTP, Tora
- **插件架构**: 动态加载，灵活切换

## 📊 性能指标

| 指标 | 数值 | 说明 |
|------|------|------|
| **Tick2Order** | < 20μs | 从收到行情到发送订单 (优化后) |
| **订单执行** | < 5μs | 插件内部处理时间 |
| **共享内存延迟** | 1-3μs | OrderServer 通信延迟 |
| **风控检查** | 2-5μs | RiskServer 检查延迟 |

## 📊 架构优势

| 特性 | 说明 |
|------|------|
| 插件化 | 动态加载柜台 API，灵活切换 |
| 双路径通信 | 共享内存 (策略) + 网络 (手动) |
| 风控集成 | 与 XRiskJudge 深度集成 |
| 状态管理 | 精确订单状态机，支持持久化 |
| 高性能 | CPU 绑定、无锁队列、批量处理 |

## 📖 相关文档

- [XRiskJudge 风控系统](../XRiskJudge/README.md) - 风控检查实现
- [XMarketCenter 行情网关](../XMarketCenter/README.md) - 行情接收
- [SHMServer 共享内存](../SHMServer/01_Architecture.md) - IPC 原理
- [部署运维指南](../../deployment/DeploymentGuide.md) - XTrader 部署

## 🔗 外部资源

- **GitHub**: https://github.com/QuantFabric/XTrader
- **CTP API 文档**: https://www.sfit.com.cn/
- **REM API 文档**: https://www.hbrz.com/
- **YD API 文档**: https://www.ydtech.com.cn/

---
作者: @scorpiostudio @yutiansut
