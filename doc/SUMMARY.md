# Summary

[介绍](README.md)

---

# 架构文档

- [系统总体架构](architecture/SystemArchitecture.md)

---

# 核心模块

## 基础组件

- [Utils - 工具库](modules/Utils/README.md)
  - [模块概览](modules/Utils/00_Overview.md)

- [XAPI - 第三方库](modules/XAPI.md)

- [SHMServer - 共享内存](modules/SHMServer/README.md)
  - [架构与原理](modules/SHMServer/01_Architecture.md)

## 行情与交易

- [XMarketCenter - 行情网关](modules/XMarketCenter/README.md)
  - [设计理念与架构](modules/XMarketCenter/01_DesignPhilosophy.md)
  - [技术实现详解](modules/XMarketCenter/02_TechnicalImplementation.md)
  - [插件开发指南](modules/XMarketCenter/03_PluginDevelopment.md)

- [XTrader - 交易网关](modules/XTrader/README.md)
  - [设计架构与原理](modules/XTrader/01_DesignArchitecture.md)
  - [技术实现详解](modules/XTrader/02_TechnicalImplementation.md)
  - [插件开发指南](modules/XTrader/03_PluginDevelopment.md)

- [XRiskJudge - 风控系统](modules/XRiskJudge/README.md)
  - [模块概览](modules/XRiskJudge/00_Overview.md)

- [XQuant - 策略平台](modules/XQuant/README.md)
  - [模块概览](modules/XQuant/00_Overview.md)

## 监控与管理

- [XWatcher - Colo 监控](modules/XWatcher/README.md)
  - [模块概览](modules/XWatcher/00_Overview.md)

- [XServer - 中间件](modules/XServer/README.md)
  - [中间件架构详解](modules/XServer/01_Middleware%20Architecture.md)

- [XMonitor - GUI 客户端](modules/XMonitor/README.md)
  - [GUI 架构与插件系统](modules/XMonitor/01_GUI_Architecture.md)

---

# 部署运维

- [部署运维指南](deployment/DeploymentGuide.md)

---

# 附录

- [术语表](glossary.md)
- [常见问题](faq.md)
