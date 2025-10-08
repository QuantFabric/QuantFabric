# QuantFabric 系统架构总览

## 系统简介

QuantFabric 是一套基于 Linux C++/Python 开发的**中高频量化交易系统**，支持中国期货和股票市场交易，具有低延迟、高可靠性、模块化设计等特点。

## 整体架构

### 部署架构

```
┌─────────────────────────────────────────────────────────────┐
│                        用户侧/公司侧                          │
│  ┌──────────┐                    ┌──────────┐               │
│  │ XMonitor │ ←─────网络─────→  │ XServer  │               │
│  │(GUI客户端)│                    │(中间件)  │               │
│  └──────────┘                    └────┬─────┘               │
└────────────────────────────────────────┼──────────────────────┘
                                         │ 网络 (跨机房)
┌────────────────────────────────────────┼──────────────────────┐
│                     Colo 交易服务器     │                       │
│                    ┌────────────────┐  │                       │
│                    │   XWatcher     │←─┘                       │
│                    │  (监控组件)     │                          │
│                    └───┬────────────┘                          │
│                        │                                       │
│        ┌───────────────┼───────────────┬──────────┐           │
│        ↓               ↓               ↓          ↓           │
│  ┌──────────┐    ┌──────────┐   ┌──────────┐ ┌────────┐      │
│  │XMarket   │    │XRiskJudge│   │ XTrader  │ │ XQuant │      │
│  │Center    │    │  (风控)   │   │ (交易)   │ │ (策略) │      │
│  └────┬─────┘    └────┬─────┘   └────┬─────┘ └───┬────┘      │
│       │               │              │           │            │
│       │ 共享内存      │ 共享内存      │ 共享内存   │            │
│       └───────────────┴──────────────┴───────────┘            │
│                                                                │
│                        ↓ 柜台 API                              │
│                   交易所 (CTP/REM/YD/OES)                      │
└────────────────────────────────────────────────────────────────┘
```

### 组件职责

| 组件 | 部署位置 | 职责 | 通信方式 |
|------|---------|------|---------|
| **XMonitor** | 用户侧 | GUI 监控客户端 | 网络 → XServer |
| **XServer** | 用户侧/公司侧 | 中间件、路由转发 | 网络 ↔ XWatcher |
| **XWatcher** | Colo 机房 | 监控、命令转发 | 网络 ↔ XServer<br>网络 ↔ 交易组件 |
| **XMarketCenter** | Colo 机房 | 行情网关 | 柜台 API → 共享内存 |
| **XRiskJudge** | Colo 机房 | 风控系统 | 共享内存 ↔ XTrader |
| **XTrader** | Colo 机房 | 交易网关 | 共享内存 ↔ 柜台 API |
| **XQuant** | Colo 机房 | 策略引擎 | 共享内存 ↔ XMarketCenter/XTrader |

## 数据流向

### 1. 行情数据流

```
柜台行情 API (CTP/REM/YD)
    ↓
XMarketCenter (行情网关)
    ├─→ 共享内存 (MarketServer) → XQuant/HFTrader
    └─→ HPPackClient → XWatcher → XServer → XMonitor
```

**关键技术**:
- **插件架构**: 动态加载不同柜台行情插件
- **无锁队列**: 行情回调 → 共享内存，避免锁竞争
- **零拷贝**: 共享内存直接访问

### 2. 交易数据流（带风控）

```
XQuant 策略信号
    ↓
OrderServer 共享内存
    ↓
XTrader 读取
    ↓
RiskServer 共享内存 → XRiskJudge 风控检查
    ↓                        ↓
    ←────────────────────────┘ 返回检查结果
    ↓
柜台交易 API (报单)
    ↓
订单回报
    ├─→ 共享内存 → XQuant
    └─→ 网络 → XWatcher → XServer → XMonitor
```

**关键流程**:
1. XQuant 计算交易信号
2. 报单请求写入 OrderServer 共享内存
3. XTrader 读取并判断是否需要风控
4. 需要风控: 发送到 XRiskJudge 检查
5. 风控通过: 调用柜台 API 报单
6. 订单回报: 更新仓位/资金，回传 XQuant

### 3. 监控数据流

```
交易组件 (XMarketCenter/XTrader/XRiskJudge)
    ↓ 网络
XWatcher
    ↓ 网络
XServer
    ↓ 网络
XMonitor (GUI 展示)
```

**监控内容**:
- 行情数据
- 订单回报
- 仓位/资金
- 风控报告
- 事件日志
- 服务器性能
- 进程状态

## 核心技术架构

### 1. 插件架构

**XMarketCenter** 和 **XTrader** 采用插件模式：

```cpp
// 动态加载插件
MarketGateWay* gateway = XPluginEngine<MarketGateWay>::LoadPlugin("libCTPMarketGateWay.so");
gateway->Run();
```

**优势**:
- 解耦柜台 API
- 灵活切换柜台
- 避免静态链接冲突

### 2. 共享内存 IPC

**SHMServer** 提供高性能 IPC：

```cpp
// Server
SHMServer<Message::PackMessage> server;
server.Start("OrderServer", cpuset);

// Client
SHMConnection<Message::PackMessage> client;
client.Start("OrderServer");
client.Send(&msg);
```

**特点**:
- SPSC 无锁队列
- 零拷贝通信
- 微秒级延迟

### 3. 网络通信

基于 **HPSocket** 框架：

```cpp
// 客户端
HPPackClient client;
client.Start(ip, port);
client.SendData(buffer, len);

// 服务端
HPPackServer server;
server.Start(ip, port);
server.SendData(connId, buffer, len);
```

**协议**: Pack 协议 (长度 + 数据)

### 4. 配置管理

基于 **YAML-CPP**：

```yaml
XTraderConfig:
  Account: 123456
  ServerIP: 127.0.0.1
  Port: 8001
  CPUSET: [10, 11]
```

**加载**:
```cpp
Utils::LoadXTraderConfig("XTrader.yml", config, errorMsg);
```

## 性能优化策略

### 1. CPU 绑定

```bash
# 隔离 CPU
isolcpus=5-20

# 绑定进程
taskset -c 5 ./XMarketCenter_0.9.3
```

**关键线程绑定**:
- XMarketCenter: CPU 5
- XRiskJudge: CPU 12
- XTrader WorkThread: CPU 10
- XTrader OrderServer: CPU 11
- XQuant: CPU 17

### 2. 无锁编程

- **LockFreeQueue**: 单进程内无锁队列
- **IPCLockFreeQueue**: 进程间无锁队列
- **parallel_flat_hash_map**: 并行哈希表

### 3. 内存优化

- **共享内存 IPC**: 零拷贝
- **环形缓冲区**: 固定内存分配
- **内存对齐**: 缓存行对齐

### 4. 网络优化

- **Solarflare Onload**: 内核旁路
- **TCPDirect**: 零拷贝网络
- **UDP 多播行情**: 低延迟

### 5. 编译优化

```cmake
set(CMAKE_CXX_FLAGS "-O3 -march=native")
add_definitions(-D_GLIBCXX_USE_CXX11_ABI=0)
```

## 延迟分析

### 多进程架构延迟 (Tick2Order)

**测试环境**: AMD EPYC 2.6GHz

| 阶段 | 延迟 | 说明 |
|------|------|------|
| 柜台回调 → XMarketCenter | 5-10μs | API 回调处理 |
| XMarketCenter → 共享内存 | 2-5μs | 无锁队列写入 |
| 共享内存 → XQuant | 2-5μs | 无锁队列读取 |
| XQuant 策略计算 | 5-10μs | 信号生成 |
| XQuant → OrderServer | 2-5μs | 共享内存写入 |
| XTrader 读取 | 2-5μs | 共享内存读取 |
| XRiskJudge 检查 | 3-8μs | 风控逻辑 |
| XTrader → 柜台 API | 5-10μs | API 调用 |
| 柜台 API 处理 | 10-20μs | 柜台内部 |
| **总计** | **50-60μs** | |

**超频优化** (4.8GHz + CPU 绑定):
- Tick2Order 中位数: **<20μs**

### HFTrader 单进程架构延迟

**测试环境**: YD 易达 + 5.0GHz CPU

| 指标 | 值 |
|------|-----|
| 中位数 | **1.1μs** |
| 最小值 | 785ns |
| 最大值 | 3.2μs |
| 99 分位 | 3.0μs |

## 容错与可靠性

### 1. 自动重连

```cpp
// XMarketCenter/XTrader 柜台断线重连
void OnFrontDisconnected(int reason) {
    FMTLOG(fmtlog::WRN, "Disconnected, reconnecting...");
    ReconnectAPI();
}
```

### 2. 共享内存恢复

```cpp
// 检测共享内存异常
if (!m_SHMConnection->IsConnected()) {
    FMTLOG(fmtlog::ERR, "SHM disconnected, restarting...");
    m_SHMConnection->Restart();
}
```

### 3. 日志记录

- **异步日志**: FMTLog/SPDLog
- **文件轮转**: 按日期自动轮转
- **日志级别**: DEBUG/INFO/WARN/ERROR

### 4. 进程监控

XWatcher 监控进程状态，异常时告警：

```cpp
if (processStatus == CRASHED) {
    SendAlarm("Process crashed: " + processName);
}
```

## 扩展性设计

### 1. 添加新柜台

1. 实现 `MarketGateWay` / `TradeGateWay` 接口
2. 编译为 `.so` 插件
3. 修改配置文件指定插件路径

### 2. 添加新策略

**C++**:
```cpp
class MyStrategy : public StrategyEngine {
    void OnMarketData(...) override;
};
```

**Python**:
```python
class MyStrategy:
    def on_market_data(self, data):
        pass
```

### 3. 支持新交易所

1. 添加交易所定义 (PackMessage.hpp)
2. 更新 Ticker 列表格式
3. 实现柜台插件

## 安全性设计

### 1. 用户权限管理

- XServer SQLite 数据库存储用户权限
- 插件级权限控制
- 消息级订阅控制

### 2. 风控系统

- 流速控制
- 撤单限制
- 防自成交
- 账户锁定

### 3. 网络安全

- 私有网络部署
- VPN 加密通信 (可选)

## 系统配置

### 推荐硬件配置

**Colo 交易服务器**:
- CPU: Intel Xeon/AMD EPYC, 主频 ≥3.0GHz
- 内存: ≥32GB
- 网络: 万兆网卡 (Solarflare 更佳)
- 操作系统: CentOS 7/8, Ubuntu 20.04

**用户侧服务器**:
- CPU: 4 核+
- 内存: 8GB+
- 网络: 千兆

### 系统参数优化

**共享内存**:
```bash
sysctl -w kernel.shmmax=17179869184
sysctl -w kernel.shmall=4194304
```

**CPU 隔离**:
```bash
# /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=5-20"
```

**网络优化**:
```bash
# /etc/sysctl.conf
net.core.rmem_max=134217728
net.core.wmem_max=134217728
```

## 运行流程

### 启动顺序

**Colo 交易服务器**:
```bash
1. ./XWatcher_0.9.3 XWatcher.yml           # 启动监控
2. ./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so  # 行情
3. ./XRiskJudge_0.9.3 XRiskJudge.yml       # 风控
4. ./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so  # 交易
5. ./XQuant_0.9.3 XQuant.yml               # 策略 (可选)
```

**用户侧**:
```bash
1. ./XServer_0.9.3 XServer.yml             # 中间件
2. ./XMonitor                              # GUI
```

### 日常运维

**查看日志**:
```bash
tail -f logs/XTrader_20250115.log
```

**查看共享内存**:
```bash
ipcs -m
```

**清理共享内存**:
```bash
ipcrm -M <key>
```

**进程监控**:
```bash
ps aux | grep XTrader
top -p <pid>
```

## 总结

QuantFabric 是一套**高性能、模块化、可扩展**的量化交易系统，通过：

1. **插件架构**: 解耦柜台 API，灵活适配
2. **共享内存 IPC**: 微秒级进程间通信
3. **无锁设计**: 避免锁竞争，提升性能
4. **CPU 绑定**: 减少调度开销
5. **网络优化**: 内核旁路、零拷贝

实现了 **20-60μs 的 Tick2Order 延迟**（多进程架构），以及 **<2μs 的极低延迟**（HFTrader 单进程架构），满足中高频量化交易需求。

**作者**: @scorpiostudio @yutiansut
**GitHub**: https://github.com/QuantFabric/QuantFabric
