# XWatcher 模块文档

## 模块概述

XWatcher 是部署在 **Colo 交易服务器**的监控组件，负责转发控制命令和交易数据，监控服务器性能和交易进程状态。

**项目地址**: [XWatcher](https://github.com/QuantFabric/XWatcher)

## 核心功能

1. **命令转发**: 转发 XServer 的控制命令到交易组件
2. **数据转发**: 转发交易数据到 XServer
3. **性能监控**: 监控服务器 CPU、内存、网络
4. **进程管理**: 启动/停止交易组件

## 架构设计

```
XServer (中间件)
    ↓ 网络
XWatcher (Colo 服务器)
    ├─> XMarketCenter
    ├─> XTrader
    ├─> XRiskJudge
    └─> XQuant
```

## 核心组件

### 1. 命令转发

**流程**:
```
XMonitor → XServer → XWatcher → XTrader
```

**命令类型**:
- 报单/撤单请求
- 风控参数修改
- 进程启动/停止

### 2. 数据转发

**流程**:
```
XTrader → XWatcher → XServer → XMonitor
```

**数据类型**:
- 行情数据
- 订单回报
- 仓位/资金
- 事件日志

### 3. 性能监控

**监控指标**:
- CPU 使用率
- 内存使用率
- 网络流量
- 磁盘 I/O

**实现**:
```cpp
void MonitorPerformance() {
    ColoStatus status;
    status.CPUUsage = GetCPUUsage();
    status.MemoryUsage = GetMemoryUsage();
    // 发送到 XServer
    SendToServer(&status);
}
```

### 4. 进程管理

**功能**:
- 启动交易组件
- 停止交易组件
- 监控进程状态

**实现**:
```cpp
void StartApp(const std::string& appName) {
    system(("./scripts/start_" + appName + ".sh").c_str());
}
```

## 配置文件

```yaml
XWatcherConfig:
  ServerIP: 192.168.1.100       # XServer IP
  ServerPort: 9000
  ColoName: Colo1               # Colo 名称
  MonitorInterval: 5            # 监控间隔 (秒)
```

## 运行方式

```bash
./XWatcher_0.9.3 XWatcher.yml
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XWatcher
