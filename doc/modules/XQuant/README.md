# XQuant 文档索引

## 📚 文档列表

### 1. [模块概览](00_Overview.md)
快速了解 XQuant 策略平台

**内容**:
- 模块概述
- C++ 策略开发
- Python 策略开发
- K 线生成器
- 策略示例
- 编译运行

**适合**:
- 策略开发者
- 量化交易员
- 快速参考

## 🎯 核心功能

### 策略开发
- **C++ 策略**: 极致低延迟 (<5μs)
- **Python 策略**: 开发效率高 (毫秒级)
- **K 线生成**: 实时 K 线计算
- **策略回测**: 历史数据回放

### 数据订阅
- **行情订阅**: 从 XMarketCenter 共享内存读取
- **订单回报**: 从 XTrader 共享内存读取
- **仓位资金**: 实时查询

### 订单执行
- **快速报单**: 通过 OrderClient 共享内存发送
- **撤单管理**: 灵活撤单策略
- **风控集成**: 自动风控检查

## 📊 策略架构

```
┌────────────────────────────┐
│        XQuant 架构         │
│  ┌──────────────────────┐ │
│  │   StrategyEngine     │ │
│  └──────────┬───────────┘ │
│             ↓              │
│  ┌──────────────────────┐ │
│  │  MarketClient (SHM)  │ │ ← XMarketCenter
│  │  OrderClient  (SHM)  │ │ → XTrader
│  └──────────────────────┘ │
│             ↓              │
│  ┌──────────────────────┐ │
│  │   C++/Python 策略    │ │
│  │  - OnTick()          │ │
│  │  - OnOrder()         │ │
│  │  - OnTrade()         │ │
│  └──────────────────────┘ │
└────────────────────────────┘
```

## 📊 性能对比

| 语言 | 延迟 | 吞吐量 | 开发效率 |
|------|------|--------|----------|
| **C++** | 1-5μs | 高 | 中 |
| **Python** | 100-500μs | 中 | 高 |

## 💡 策略示例

### C++ 策略
```cpp
class MyStrategy : public Strategy {
    void OnTick(const MarketData::TFutureMarketData& tick) override {
        // 行情回调
        if (tick.LastPrice > m_UpperBound) {
            // 卖出
            SendOrder(tick.Ticker, 'S', tick.AskPrice1, 1);
        }
    }

    void OnOrder(const Message::TOrderStatus& order) override {
        // 订单回调
        if (order.OrderStatus == EORDER_ALL_TRADED) {
            Logger::Info("Order filled: {}", order.OrderRef);
        }
    }
};
```

### Python 策略
```python
class MyStrategy(Strategy):
    def on_tick(self, tick):
        # 行情回调
        if tick.last_price > self.upper_bound:
            # 卖出
            self.send_order(tick.ticker, 'S', tick.ask_price1, 1)

    def on_order(self, order):
        # 订单回调
        if order.status == 'ALL_TRADED':
            print(f"Order filled: {order.order_ref}")
```

## 📖 相关文档

- [XMarketCenter 行情网关](../XMarketCenter/README.md) - 行情数据源
- [XTrader 交易网关](../XTrader/README.md) - 订单执行
- [SHMServer 共享内存](../SHMServer/01_Architecture.md) - 策略通信
- [部署运维指南](../../deployment/DeploymentGuide.md) - XQuant 部署

---
作者: @scorpiostudio @yutiansut
