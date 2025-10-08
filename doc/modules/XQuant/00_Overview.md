# XQuant 模块文档

## 模块概述

XQuant 是 QuantFabric 的**策略交易平台**，提供 C++ 和 Python 版本，通过共享内存与 XMarketCenter、XTrader、XRiskJudge 进行高速通信。

**项目地址**: [XQuant](https://github.com/QuantFabric/XQuant)

## 核心功能

1. **行情接收**: 从 XMarketCenter 读取实时行情
2. **策略计算**: 根据行情触发交易信号
3. **订单发送**: 将报单/撤单请求发送到 XTrader
4. **回报处理**: 接收订单回报、仓位、资金信息
5. **K线生成**: 内置 K 线生成器

## 架构设计

### 数据流向

```
XMarketCenter (MarketServer)
        ↓ 共享内存
    XQuant 策略引擎
        ├─> 计算信号
        ↓ 共享内存
XTrader (OrderServer)
        ↓ 柜台 API
      交易所
```

### 版本对比

| 特性 | C++ 版本 | Python 版本 |
|------|---------|------------|
| 延迟 | 低 (微秒级) | 中 (毫秒级) |
| 开发效率 | 中 | 高 |
| 适用场景 | 中高频策略 | 中频策略 |
| 依赖 | SHMServer C++ | SHMServer Python 绑定 |

## 核心组件 (C++ 版本)

### 1. QuantEngine (策略引擎)

**文件**: `QuantEngine.cpp`

**主要职责**:
- 连接 MarketServer (行情共享内存)
- 连接 OrderServer (订单共享内存)
- 连接 XWatcher (监控)
- 管理策略实例

**启动流程**:
```cpp
1. LoadConfig()           // 加载配置
2. Run()
   - 连接 MarketServer
   - 连接 OrderServer
   - 连接 XWatcher
   - 初始化策略
   - 启动工作线程
```

### 2. StrategyEngine (策略基类)

**文件**: `StrategyEngine.hpp`

**抽象接口**:
```cpp
class StrategyEngine {
public:
    virtual void OnMarketData(const MarketData::TFutureMarketData& data) = 0;
    virtual void OnOrderStatus(const Message::TOrderStatus& status) = 0;
    virtual void OnAccountPosition(const Message::TAccountPosition& position) = 0;
    virtual void OnAccountFund(const Message::TAccountFund& fund) = 0;
protected:
    void SendOrder(const Message::TOrderRequest& order);    // 发送报单
    void CancelOrder(const Message::TActionRequest& action); // 发送撤单
};
```

### 3. KLineGenerator (K线生成器)

**文件**: `KLineGenerator.hpp`

**功能**:
- 从 Tick 生成多周期 K 线
- 支持任意周期 (60s, 300s, 3600s 等)
- K 线完成时回调

**使用示例**:
```cpp
// 初始化 (1分钟和5分钟)
KLineGenerator kline("rb2505", 0, 2, {60, 300});
kline.SetOnWindowBarFunc(OnKLineCallback);

// 处理 Tick
kline.ProcessTick(startTime, endTime, timestamp, price, volume);

// 获取历史 K 线
std::vector<BarData>& bars_1min = kline.GetHistory(60);
```

### 4. 共享内存连接

**MarketServer 连接**:
```cpp
SHMIPC::SHMConnection<MarketData::TFutureMarketData, ClientConf> m_MarketConn;
m_MarketConn.Start("MarketServer");

// 接收行情
MarketData::TFutureMarketData data;
if (m_MarketConn.Receive(&data, sizeof(data))) {
    OnMarketData(data);
}
```

**OrderServer 连接**:
```cpp
SHMIPC::SHMConnection<Message::PackMessage, ClientConf> m_OrderConn;
m_OrderConn.Start("OrderServer" + Account);

// 发送报单
Message::PackMessage msg;
msg.MessageType = Message::EOrderRequest;
// 填充订单信息
m_OrderConn.Send(&msg, sizeof(msg));

// 接收回报
if (m_OrderConn.Receive(&msg, sizeof(msg))) {
    OnOrderStatus(msg.OrderStatus);
}
```

## 策略示例 (C++)

### TestStrategy.hpp

```cpp
class TestStrategy : public StrategyEngine {
public:
    void OnMarketData(const MarketData::TFutureMarketData& data) override {
        // 简单均线策略
        m_PriceSum += data.LastPrice;
        m_Count++;
        double avgPrice = m_PriceSum / m_Count;

        if (data.LastPrice > avgPrice * 1.01 && m_Position == 0) {
            // 价格突破均线 1%，买入
            Message::TOrderRequest order;
            strcpy(order.Ticker, data.Ticker);
            order.Direction = Message::EBUY;
            order.OrderPrice = data.AskPrice1;
            order.OrderVolume = 1;
            SendOrder(order);
        }
    }

    void OnOrderStatus(const Message::TOrderStatus& status) override {
        if (status.OrderStatus == Message::EALLTRADED) {
            m_Position += status.TotalTradedVolume;
        }
    }

private:
    double m_PriceSum = 0.0;
    int m_Count = 0;
    int m_Position = 0;
};
```

## Python 版本

### 依赖

使用 SHMServer Python 绑定:
```python
import shm_connection
import pack_message
```

### 策略示例 (SMAStrategy.py)

```python
class SMAStrategy:
    def __init__(self, account, ticker):
        self.account = account
        self.ticker = ticker
        self.prices = []
        self.position = 0

        # 连接行情
        self.market_conn = shm_connection.SHMConnection(account)
        self.market_conn.start("MarketServer")

        # 连接订单
        self.order_conn = shm_connection.SHMConnection(account)
        self.order_conn.start("OrderServer" + account)

    def on_market_data(self, data):
        # 简单移动平均策略
        self.prices.append(data['LastPrice'])
        if len(self.prices) > 20:
            self.prices.pop(0)

        sma = sum(self.prices) / len(self.prices)

        if data['LastPrice'] > sma * 1.01 and self.position == 0:
            # 买入信号
            self.send_order(data['AskPrice1'], 1, 'BUY')

    def send_order(self, price, volume, direction):
        order = pack_message.OrderRequest()
        order.Ticker = self.ticker
        order.OrderPrice = price
        order.OrderVolume = volume
        order.Direction = direction
        self.order_conn.send(order)

    def run(self):
        while True:
            data = self.market_conn.receive()
            if data:
                self.on_market_data(data)
```

### 运行 Python 策略

```bash
# 设置 Python 路径
export PYTHONPATH=/path/to/SHMServer/pybind11/build:$PYTHONPATH

# 运行策略
python3 SMAStrategy.py
```

## 配置文件

### XQuant.yml

```yaml
XQuantConfig:
  XWatcherIP: 127.0.0.1
  XWatcherPort: 8001
  StrategyName: TestStrategy
  AccountList: 188795, 237477
  TickerListPath: /path/to/TickerList.yml
  MarketServerName: MarketServer
  OrderServerName: OrderServer
  CPUSET: 17                    # CPU 绑定
  SnapshotInterval: 0           # 快照间隔 (秒)
  SlicePerSec: 2                # 每秒 Slice 数
  KLineIntervals: 60, 300       # K线周期 (秒)
  TradingSection:
    - 21:00:00-23:30:00         # 夜盘
    - 09:00:00-10:15:00         # 日盘
    - 10:30:00-11:30:00
    - 13:30:00-15:00:00
```

## 性能指标

### 多进程架构延迟

基于 XMarketCenter + XRiskJudge + XTrader + XQuant (C++):

| CPU | Tick2Order 中位数 |
|-----|------------------|
| AMD EPYC 2.6GHz | 50-60μs |
| 超频 4.8GHz + CPU绑定 | <20μs |

**延迟分解**:
- XMarketCenter 处理: 5-10μs
- 共享内存传输: 2-5μs
- XQuant 计算: 5-10μs
- XRiskJudge 检查: 3-8μs
- XTrader 报单: 5-10μs
- 柜台 API: 10-20μs

### Python 版本延迟

- Tick2Order: 1-5ms
- 适合中频策略 (秒级交易)

## 编译与运行

### C++ 编译

```bash
cd QuantFabric
sh build_release.sh
# 输出: build/XQuant_0.9.3
```

### 运行 C++ 策略

```bash
# 基本启动
./XQuant_0.9.3 XQuant.yml

# CPU 绑定
taskset -c 17 ./XQuant_0.9.3 XQuant.yml
```

### Python 策略

```bash
# 编译 SHMServer Python 绑定
cd SHMServer/pybind11/build
cmake .. && make

# 运行策略
export PYTHONPATH=/path/to/SHMServer/pybind11/build:$PYTHONPATH
python3 SMAStrategy.py
```

## 开发策略

### C++ 策略开发步骤

1. **继承 StrategyEngine**:
```cpp
class MyStrategy : public StrategyEngine {
    void OnMarketData(...) override;
    void OnOrderStatus(...) override;
};
```

2. **实现回调函数**:
```cpp
void MyStrategy::OnMarketData(const MarketData::TFutureMarketData& data) {
    // 策略逻辑
}
```

3. **注册到 QuantEngine**:
```cpp
QuantEngine engine;
MyStrategy* strategy = new MyStrategy();
engine.RegisterStrategy(strategy);
engine.Run();
```

### Python 策略开发步骤

1. **导入模块**:
```python
import shm_connection
import pack_message
```

2. **实现策略类**:
```python
class MyStrategy:
    def on_market_data(self, data):
        # 策略逻辑
        pass
```

3. **连接共享内存并运行**:
```python
strategy = MyStrategy()
strategy.run()
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XQuant
