# XRiskJudge 模块文档

## 模块概述

XRiskJudge 是 QuantFabric 交易系统的**风控系统**，提供账户间风控功能，包括流速控制、撤单限制、防自成交、账户锁定等。

**项目地址**: [XRiskJudge](https://github.com/QuantFabric/XRiskJudge)

## 核心功能

1. **流速控制**: 限制每秒报单数量
2. **撤单限制**: 限制 Ticker 和订单级别的撤单次数
3. **防自成交**: 检测同账户对手盘
4. **账户锁定**: 紧急情况下锁定账户
5. **订单管理**: 维护订单状态，统计撤单次数

## 架构设计

### 数据流向

```
XTrader → RiskServer (共享内存) → XRiskJudge
                                       ↓
                              风控检查 (流速/撤单/自成交)
                                       ↓
XTrader ← RiskServer (共享内存) ← 检查结果
```

### 风控检查流程

```
1. 接收报单/撤单请求
   └─> 从 RiskServer 共享内存读取

2. 账户锁定检查
   └─> 如果账户锁定，拒绝

3. 流速控制检查
   └─> 统计时间窗口内报单数

4. 撤单限制检查
   ├─> Ticker 级别撤单次数
   └─> 订单级别撤单次数

5. 防自成交检查
   └─> 检查同账户对手盘

6. 返回检查结果
   ├─> PASS: 通过
   └─> NOPASS: 拒绝
```

## 核心组件

### 1. RiskJudgeEngine (风控引擎)

**主要职责**:
- 加载风控参数
- 接收 XTrader 风控请求
- 执行风控检查
- 返回检查结果
- 管理订单状态

**关键数据结构**:
```cpp
// 风控参数
struct TRiskJudgeSettings {
    int FlowLimit;              // 流速限制 (笔/秒)
    int TickerCancelLimit;      // Ticker 撤单限制
    int OrderCancelLimit;       // 订单撤单限制
    bool LockAccount;           // 账户锁定
    bool CheckSelfTrade;        // 检查自成交
};

// 撤单统计
phmap::parallel_flat_hash_map<std::string, int> m_TickerCancelCountMap;  // Ticker 撤单计数
phmap::parallel_flat_hash_map<int, int> m_OrderCancelCountMap;          // 订单撤单计数
```

### 2. RiskServer (共享内存服务)

**功能**:
- 接收 XTrader 的风控请求
- 返回风控检查结果

**实现**:
```cpp
class RiskServer : public SHMIPC::SHMServer<Message::PackMessage, ServerConf> {
    void OnMessage(Message::PackMessage& request) {
        // 执行风控检查
        CheckRisk(request);
        // 返回结果
        SendResponse(request);
    }
};
```

### 3. HPPackClient (监控客户端)

**功能**:
- 连接 XWatcher
- 接收风控参数更新命令
- 发送风控报告和事件日志

## 风控策略

### 1. 流速控制

```cpp
// 时间窗口内报单计数
if (m_OrderCountInWindow >= m_RiskSettings.FlowLimit) {
    // 拒绝报单
    request.RiskStatus = Message::ECHECKED_NOPASS;
    request.ErrorMsg = "Flow limit exceeded";
}
```

**配置示例**:
```yaml
RiskJudgeConfig:
  FlowLimit: 10    # 每秒最多 10 笔
```

### 2. Ticker 撤单限制

```cpp
// 统计 Ticker 当日撤单次数
int cancelCount = m_TickerCancelCountMap[ticker];
if (cancelCount >= m_RiskSettings.TickerCancelLimit) {
    // 拒绝撤单
    request.RiskStatus = Message::ECHECKED_NOPASS;
    request.ErrorMsg = fmt::format("{} cancel limit exceeded", ticker);
}
```

**配置示例**:
```yaml
RiskJudgeConfig:
  TickerCancelLimit: 100    # 每个 Ticker 每天最多撤单 100 次
```

### 3. 订单撤单限制

```cpp
// 统计订单撤单次数
int orderCancelCount = m_OrderCancelCountMap[orderToken];
if (orderCancelCount >= m_RiskSettings.OrderCancelLimit) {
    // 拒绝撤单
    request.RiskStatus = Message::ECHECKED_NOPASS;
}
```

**配置示例**:
```yaml
RiskJudgeConfig:
  OrderCancelLimit: 5    # 每个订单最多撤单 5 次
```

### 4. 防自成交

```cpp
// 检查待成交订单中是否有对手盘
for (auto& order : m_PendingOrders) {
    if (order.Account == newOrder.Account &&
        order.Direction != newOrder.Direction &&
        order.Price == newOrder.Price) {
        // 检测到自成交风险
        request.RiskStatus = Message::ECHECKED_NOPASS;
        request.ErrorMsg = "Self-trade detected";
    }
}
```

### 5. 账户锁定

```cpp
if (m_RiskSettings.LockAccount) {
    // 拒绝所有报单/撤单
    request.RiskStatus = Message::ECHECKED_NOPASS;
    request.ErrorMsg = "Account locked";
}
```

**通过 XMonitor 设置**:
- GUI 界面直接锁定/解锁账户

## 配置文件

### XRiskJudge.yml

```yaml
XRiskJudgeConfig:
  Account: 123456
  ServerIP: 127.0.0.1        # XWatcher IP
  Port: 8001
  RiskServerName: RiskServer # 共享内存名称
  CPUSET: 12                 # CPU 绑定
  RiskJudge:
    FlowLimit: 10            # 流速限制
    TickerCancelLimit: 100   # Ticker 撤单限制
    OrderCancelLimit: 5      # 订单撤单限制
    LockAccount: false       # 账户锁定
    CheckSelfTrade: true     # 防自成交
```

## 风控报告

### TRiskReport

```cpp
struct TRiskReport {
    char Account[16];
    char Ticker[32];
    int TickerCancelCount;     // Ticker 当日撤单次数
    int TodayOrderCount;       // 当日报单总数
    int TodayFlowCount;        // 当前时间窗口报单数
    bool LockStatus;           // 账户锁定状态
};
```

**发送到 XMonitor**:
- 实时显示撤单统计
- 显示账户锁定状态

## 编译与运行

### 编译

```bash
cd QuantFabric
sh build_release.sh
# 输出: build/XRiskJudge_0.9.3
```

### 运行

```bash
# 基本启动
./XRiskJudge_0.9.3 XRiskJudge.yml

# CPU 绑定
taskset -c 12 ./XRiskJudge_0.9.3 XRiskJudge.yml
```

## 动态风控参数调整

### 通过 XMonitor 修改

XMonitor 发送风控参数更新命令:
```cpp
Message::TRiskControl riskControl;
riskControl.Account = "123456";
riskControl.FlowLimit = 20;          // 调整流速
riskControl.TickerCancelLimit = 200; // 调整撤单限制
// 发送到 XRiskJudge
```

XRiskJudge 接收并更新:
```cpp
void OnRiskControl(const Message::TRiskControl& control) {
    m_RiskSettings.FlowLimit = control.FlowLimit;
    m_RiskSettings.TickerCancelLimit = control.TickerCancelLimit;
    // 记录日志
    FMTLOG(fmtlog::INF, "Update risk params: FlowLimit={}", control.FlowLimit);
}
```

## 监控与调试

### 日志

```bash
# 风控拒单日志
[WRN] CheckFlowLimit FAILED, Account:123456 FlowCount:11 > Limit:10
[WRN] CheckTickerCancelLimit FAILED, rb2505 CancelCount:101 > Limit:100
[INF] CheckRisk PASS, OrderToken:12345
```

### 风控报告查看

通过 XMonitor RiskJudge 插件查看:
- 各 Ticker 撤单次数
- 当日报单统计
- 账户锁定状态

## 性能优化

1. **并行哈希表**: 使用 `phmap::parallel_flat_hash_map` 减少锁竞争
2. **共享内存 IPC**: 低延迟通信
3. **CPU 绑定**: 避免调度开销

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XRiskJudge
