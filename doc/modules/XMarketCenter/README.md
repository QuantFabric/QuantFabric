# XMarketCenter 文档索引

## 📚 文档列表

### 1. [设计理念与架构](01_DesignPhilosophy.md)
深入理解 XMarketCenter 的设计思想

**内容**:
- 核心设计目标（极致低延迟、高可靠性、高扩展性）
- 架构分层设计
- 数据流向设计
- 性能优化策略
- 设计模式应用（插件模式、生产者-消费者、发布-订阅）
- 可靠性保证机制
- 监控与诊断
- 设计权衡分析

**适合**:
- 系统架构师
- 核心开发者
- 想深入理解系统设计的读者

### 2. [技术实现详解](02_TechnicalImplementation.md)
核心技术的具体实现

**内容**:
- 插件加载机制（dlopen、符号导出）
- 无锁队列原理（SPSC、内存序、环形缓冲）
- 共享内存发布（SHMServer 架构、内存布局）
- 网络通信机制（HPSocket Pack 协议）
- 延迟优化技术（CPU 缓存、编译器优化、系统调用优化）

**适合**:
- C++ 开发者
- 性能优化工程师
- 想学习无锁编程的读者

### 3. [插件开发指南](03_PluginDevelopment.md)
手把手教你开发行情插件

**内容**:
- 完整的插件开发流程（6 个步骤）
- 插件接口定义与实现
- 配置加载与管理
- CMake 编译配置
- 插件开发最佳实践
- 常见问题排查

**适合**:
- 插件开发者
- 需要对接新柜台的工程师
- 想扩展系统功能的开发者

### 4. [基础文档 - 模块概览](../XMarketCenter.md)
快速了解 XMarketCenter

**内容**:
- 模块概述
- 核心功能
- 配置文件
- 编译运行
- 基础示例

**适合**:
- 初学者
- 快速查阅参考

## 🎯 学习路径

### 新手入门
1. 先阅读 [模块概览](../XMarketCenter.md) 了解基本概念
2. 查看配置文件和运行方式
3. 运行 Demo 体验功能

### 深入学习
1. 阅读 [设计理念与架构](01_DesignPhilosophy.md) 理解设计思想
2. 学习 [技术实现详解](02_TechnicalImplementation.md) 掌握核心技术
3. 结合源码深入分析

### 插件开发
1. 阅读 [插件开发指南](03_PluginDevelopment.md)
2. 参考 CTP 插件实现
3. 开发自己的插件

## 📖 相关文档

- [SHMServer 架构](../SHMServer/01_Architecture.md) - 共享内存原理
- [系统总体架构](../../architecture/SystemArchitecture.md) - 整体设计
- [部署运维指南](../../deployment/DeploymentGuide.md) - 部署配置

## 🔗 外部资源

- **GitHub**: https://github.com/QuantFabric/XMarketCenter
- **CTP API 文档**: https://www.sfit.com.cn/
- **HPSocket 文档**: https://github.com/ldcsaa/HP-Socket
