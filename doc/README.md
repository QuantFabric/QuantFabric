# QuantFabric 项目文档

欢迎查阅 QuantFabric 量化交易系统完整文档！

## 📚 在线文档

本文档已使用 mdBook 构建为在线文档，提供更好的阅读体验：

- **构建文档**: `cd doc && mdbook build`
- **本地预览**: `cd doc && mdbook serve`
- **访问地址**: http://localhost:3000

生成的静态文件位于 `doc/book/` 目录，可部署到任何静态网站服务器。

---

## 文档目录

> 📖 **标注此图标的模块提供深度技术文档**，包含设计理念、实现原理、开发指南等详细内容

### 📚 架构文档

- **[系统总体架构](architecture/SystemArchitecture.md)**
  - 整体部署架构
  - 数据流向分析
  - 核心技术架构
  - 性能优化策略
  - 延迟分析与优化

### 🔧 模块文档

#### 基础模块

- **[Utils - 基础工具库](modules/Utils/README.md)** 📖
  - 配置管理 (YMLConfig)
  - 网络通信 (HPPackClient/Server)
  - 进程间通信 (IPC)
  - 日志系统 (Logger)
  - 消息协议 (PackMessage)
  - 工具函数 (时间、字符串、CPU绑定)

- **[XAPI - 第三方库](modules/XAPI.md)**
  - 基础库 (YAML-CPP, SPDLog, HPSocket)
  - 期货柜台 API (CTP, REM, YD)
  - 股票柜台 API (OES, XTP, Tora)
  - 网络优化库 (Onload, TCPDirect)

- **[SHMServer - 共享内存通信](modules/SHMServer/README.md)** 📖
  - [架构与原理深入](modules/SHMServer/01_Architecture.md) - 无锁队列、共享内存设计、性能优化
  - 共享内存 IPC (1-5μs 极致低延迟)
  - SPSC 无锁队列 (原子操作、内存序)
  - Python 绑定 (pybind11)

#### 交易组件

- **[XMarketCenter - 行情网关](modules/XMarketCenter/README.md)** 📖
  - [设计理念与架构](modules/XMarketCenter/01_DesignPhilosophy.md) - 设计目标、架构分层、数据流向、设计模式
  - [技术实现详解](modules/XMarketCenter/02_TechnicalImplementation.md) - 插件加载、无锁队列、共享内存、性能优化
  - [插件开发指南](modules/XMarketCenter/03_PluginDevelopment.md) - 完整开发流程、示例代码、最佳实践
  - 极致低延迟 (<15μs)
  - CTP/REM/YD 插件实现

- **[XTrader - 交易网关](modules/XTrader/README.md)** 📖
  - [设计架构与原理](modules/XTrader/01_DesignArchitecture.md) - 架构设计、数据流向、线程模型、状态管理
  - [技术实现详解](modules/XTrader/02_TechnicalImplementation.md) - 插件加载、共享内存、风控交互、订单管理
  - [插件开发指南](modules/XTrader/03_PluginDevelopment.md) - 交易插件开发流程、示例代码
  - Tick2Order < 20μs
  - CTP/REM/YD/OES 插件实现

- **[XRiskJudge - 风控系统](modules/XRiskJudge/README.md)** 📖
  - 流速控制（时间窗口计数）
  - 撤单限制（撤单比例检查）
  - 防自成交（订单簿匹配）
  - 账户锁定（异常管理）

- **[XQuant - 策略平台](modules/XQuant/README.md)** 📖
  - C++ 策略开发（微秒级延迟）
  - Python 策略开发（高开发效率）
  - K 线生成器（实时计算）
  - 策略示例与回测

#### 监控组件

- **[XWatcher - Colo 监控](modules/XWatcher/README.md)** 📖
  - 命令转发（XServer → Colo）
  - 数据转发（Colo → XServer）
  - 性能监控（CPU、内存、延迟）
  - 进程管理（启动/停止/重启）

- **[XServer - 中间件](modules/XServer/README.md)** 📖
  - [中间件架构详解](modules/XServer/01_Middleware%20Architecture.md) - 路由引擎、权限管理、消息队列、数据回放
  - 智能消息路由 (基于 Colo/Account)
  - 集中式用户权限管理
  - 历史数据回放与时间模拟

- **[XMonitor - GUI 客户端](modules/XMonitor/README.md)** 📖
  - [GUI 架构与插件系统](modules/XMonitor/01_GUI_Architecture.md) - 插件接口、管理器、可拖拽布局、性能优化
  - Qt 插件化架构 (8 大核心插件)
  - 可定制拖拽布局 (QDockWidget)
  - 高性能虚拟列表 (支持海量数据)
  - 实时行情监控与订单管理

### 🚀 部署运维

- **[部署运维指南](deployment/DeploymentGuide.md)**
  - 环境准备
  - 编译部署
  - 配置文件详解
  - 系统优化 (CPU隔离、共享内存、网络优化)
  - 启动流程
  - 进程管理
  - 监控调试
  - 常见问题排查
  - 备份恢复
  - 升级更新

## 📖 文档导航指南

### 根据角色选择文档

**系统架构师 / 核心开发者**:
1. [系统总体架构](architecture/SystemArchitecture.md) - 了解整体设计
2. [XMarketCenter 设计理念](modules/XMarketCenter/01_DesignPhilosophy.md) - 行情系统设计思想
3. [SHMServer 架构原理](modules/SHMServer/01_Architecture.md) - 共享内存 IPC 深入
4. [XServer 中间件架构](modules/XServer/01_Middleware%20Architecture.md) - 路由与权限设计

**插件开发者**:
1. [XMarketCenter 插件开发指南](modules/XMarketCenter/03_PluginDevelopment.md) - 行情插件开发
2. [XMarketCenter 技术实现](modules/XMarketCenter/02_TechnicalImplementation.md) - 插件加载机制

**GUI 开发者**:
1. [XMonitor GUI 架构](modules/XMonitor/01_GUI_Architecture.md) - Qt 插件系统

**性能优化工程师**:
1. [XMarketCenter 技术实现](modules/XMarketCenter/02_TechnicalImplementation.md) - 无锁队列、性能优化
2. [SHMServer 架构原理](modules/SHMServer/01_Architecture.md) - 共享内存优化
3. [系统总体架构 - 延迟优化](architecture/SystemArchitecture.md#延迟分析与优化)

**运维工程师**:
1. [部署运维指南](deployment/DeploymentGuide.md) - 完整部署流程
2. [系统总体架构](architecture/SystemArchitecture.md) - 了解部署架构

**初学者**:
1. 先阅读各模块的 README.md 快速了解功能
2. 再深入阅读感兴趣模块的详细文档

## 快速开始

### 1. 编译系统

```bash
git clone --recurse-submodules git@github.com:QuantFabric/QuantFabric.git
cd QuantFabric
sh build_release.sh
```

### 2. 配置文件

参考 [部署运维指南](deployment/DeploymentGuide.md) 配置各组件的 YAML 文件。

### 3. 启动交易系统

**Colo 服务器**:
```bash
./XWatcher_0.9.3 XWatcher.yml &
./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so &
./XRiskJudge_0.9.3 XRiskJudge.yml &
./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so &
./XQuant_0.9.3 XQuant.yml &
```

**用户侧**:
```bash
./XServer_0.9.3 XServer.yml &
./XMonitor
```

## 系统架构概览

```
用户侧: XMonitor ↔ XServer
            ↓ 网络
Colo侧: XWatcher ↔ XMarketCenter/XRiskJudge/XTrader/XQuant
            ↓ 共享内存
        交易策略 ↔ 柜台 API ↔ 交易所
```

## 性能指标

### 多进程架构
- **Tick2Order 延迟**: 50-60μs (AMD EPYC 2.6GHz)
- **优化后**: <20μs (超频 4.8GHz + CPU绑定)

### HFTrader 单进程架构
- **Tick2Order 中位数**: 1.1μs (YD 易达 + 5.0GHz)
- **最小延迟**: 785ns

## 核心特性

### 1. 高性能
- 共享内存 IPC (微秒级)
- 无锁队列设计
- CPU 绑定优化
- 网络内核旁路 (Onload)

### 2. 模块化
- 插件架构 (行情/交易网关)
- 组件解耦
- 灵活扩展

### 3. 多柜台支持
- **期货**: CTP, REM, YD
- **股票**: OES, XTP, Tora

### 4. 多语言策略
- **C++**: 低延迟 (微秒级)
- **Python**: 开发效率高 (毫秒级)

## 技术栈

- **语言**: C++17, Python 3
- **构建**: CMake, qmake
- **网络**: HPSocket
- **配置**: YAML-CPP
- **日志**: SPDLog, FMTLog
- **IPC**: 共享内存 + 无锁队列
- **GUI**: Qt 5

## 文档贡献

欢迎贡献文档！如发现错误或需要补充，请提交 PR。

## 技术支持

- **QQ 群**: 748930268 (验证码: QuantFabric)
- **GitHub**: https://github.com/QuantFabric/QuantFabric
- **作者**: @scorpiostudio @yutiansut

## 版本信息

- **当前版本**: 0.9.3
- **更新日期**: 2025-01-15
- **文档版本**: 1.0

---

**注意**: 本系统仅供学习和研究使用，实盘交易请充分测试并自负风险。
