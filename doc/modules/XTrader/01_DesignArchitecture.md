# XTrader 设计架构与原理

## 设计理念

### 核心定位

XTrader 是 QuantFabric 的**交易执行引擎**，负责订单全生命周期管理。

**设计目标**:
1. **极致低延迟**: Tick2Order < 20μs，订单执行 < 5μs
2. **高可靠性**: 订单不丢失、状态精确管理、异常恢复
3. **多柜台支持**: 统一接口对接 CTP/REM/YD/OES 等
4. **风控集成**: 与 XRiskJudge 深度集成，订单前置检查
5. **易扩展**: 插件架构，轻松添加新柜台

### 架构设计图

```
┌─────────────────────────────────────────────────────────┐
│                    XTrader 架构                          │
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │              TraderEngine (交易引擎)             │  │
│  │  ┌────────────────┐      ┌──────────────────┐   │  │
│  │  │  WorkThread    │      │  MonitorThread   │   │  │
│  │  │  (策略报单)     │      │  (手动报单)       │   │  │
│  │  └───────┬────────┘      └────────┬─────────┘   │  │
│  └──────────┼─────────────────────────┼────────────┘  │
│             ↓                         ↓               │
│  ┌──────────────────┐      ┌──────────────────┐      │
│  │  OrderServer     │      │  HPPackClient    │      │
│  │  (共享内存服务)   │      │  (网络客户端)     │      │
│  └──────────────────┘      └──────────────────┘      │
│             ↓                         ↓               │
│  ┌─────────────────────────────────────────────────┐  │
│  │          风控检查 (RiskServer)                   │  │
│  └─────────────────────────────────────────────────┘  │
│             ↓                                          │
│  ┌─────────────────────────────────────────────────┐  │
│  │          TradeGateWay (插件层)                   │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │  │
│  │  │   CTP    │ │   REM    │ │    YD    │ ...    │  │
│  │  └──────────┘ └──────────┘ └──────────┘        │  │
│  └─────────────────────────────────────────────────┘  │
│             ↓                                          │
│  ┌─────────────────────────────────────────────────┐  │
│  │          柜台 API (CTP/REM/YD/OES)               │  │
│  └─────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

## 架构分层设计

### 1. 应用层 (TraderEngine)

**职责**:
- 系统初始化与配置加载
- 线程管理与协调
- 插件加载与管理
- 组件间通信协调

**关键组件**:
```cpp
class TraderEngine {
    // 配置
    Utils::YAMLConfig m_YAMLConfig;

    // 插件
    TradeGateWay* m_TradeGateWay;

    // 通信组件
    HPPackClient* m_HPPackClient;      // 网络通信
    SHMConnection* m_RiskClient;       // 风控通信
    OrderServer* m_pOrderServer;       // 策略通信

    // 线程
    pthread_t m_WorkThreadID;          // 工作线程
    pthread_t m_MonitorThreadID;       // 监控线程

    // 消息队列
    LockFreeQueue<Message::PackMessage> m_RequestMessageQueue;  // 请求队列
    LockFreeQueue<Message::PackMessage> m_ReportMessageQueue;   // 回报队列
};
```

### 2. 插件层 (TradeGateWay)

**抽象接口设计**:
```cpp
class TradeGateWay {
public:
    // ========== 生命周期管理 ==========
    virtual void LoadAPIConfig() = 0;           // 加载配置
    virtual void CreateTraderAPI() = 0;         // 创建 API
    virtual void ReqUserLogin() = 0;            // 登录柜台
    virtual void Release() = 0;                 // 释放资源

    // ========== 交易接口 ==========
    virtual void ReqInsertOrder(const Message::TOrderRequest&) = 0;    // 报单
    virtual void ReqCancelOrder(const Message::TActionRequest&) = 0;   // 撤单

    // ========== 查询接口 ==========
    virtual int ReqQryFund() = 0;               // 查询资金
    virtual int ReqQryPoistion() = 0;           // 查询仓位
    virtual int ReqQryOrder() = 0;              // 查询订单
    virtual int ReqQryTrade() = 0;              // 查询成交

    // ========== 状态管理 ==========
    virtual void UpdatePosition(...) = 0;       // 更新仓位
    virtual void UpdateFund(...) = 0;           // 更新资金
    virtual void UpdateOrderStatus(...) = 0;    // 更新订单状态

    // ========== 工具方法 ==========
    virtual void SetLogger(Logger*) = 0;        // 设置日志
    virtual void SetTraderConfig(TraderConfig*) = 0;  // 设置配置
};
```

**插件继承体系**:
```
TradeGateWay (基类)
    ├── FutureTradeGateWay (期货基类)
    │   ├── CTPTradeGateWay
    │   ├── REMTradeGateWay
    │   └── YDTradeGateWay
    └── StockTradeGateWay (股票基类)
        ├── OESTradeGateWay
        ├── XTPTradeGateWay
        └── ToraTradeGateWay
```

### 3. 通信层

#### 3.1 共享内存通信 (OrderServer)

**用途**: 接收策略 (XQuant) 的高频报单/撤单请求

**技术实现**:
```cpp
class OrderServer : public SHMIPC::SHMServer<Message::PackMessage, ServerConf> {
public:
    void Start(const std::string& name, int cpuset) {
        // 创建共享内存
        CreateSharedMemory(name);

        // 绑定 CPU
        Utils::ThreadBind(pthread_self(), cpuset);

        // 启动接收线程
        StartReceiveThread();
    }

    void OnMessage(Message::PackMessage& msg) {
        // 推送到请求队列
        m_RequestMessageQueue.Push(msg);
    }
};
```

**性能特点**:
- **零拷贝**: 直接从共享内存读取
- **SPSC 无锁队列**: 单生产者单消费者
- **延迟**: 1-3μs

#### 3.2 网络通信 (HPPackClient)

**用途**:
- 连接 XWatcher，接收手动报单
- 推送订单回报到监控系统

**协议**: HPSocket Pack 协议
```cpp
class HPPackClient : public IPackClientListener {
public:
    virtual EnHandleResult OnConnect(IClient* pClient, CONNID dwConnID) {
        // 连接成功，注册客户端
        RegisterClient();
    }

    virtual EnHandleResult OnReceive(IClient* pClient, CONNID dwConnID,
                                     const BYTE* pData, int iLength) {
        // 接收手动报单/撤单
        Message::PackMessage msg;
        memcpy(&msg, pData, sizeof(msg));

        // 推送到请求队列
        m_RequestMessageQueue.Push(msg);
    }

    void SendMessage(const Message::PackMessage& msg) {
        // 发送订单回报
        pClient->Send((BYTE*)&msg, sizeof(msg));
    }
};
```

#### 3.3 风控通信 (RiskServer)

**用途**: 与 XRiskJudge 进行订单前置检查

**实现**:
```cpp
SHMConnection* m_RiskClient = new SHMConnection("RiskServer");

// 发送风控请求
void CheckRisk(Message::PackMessage& msg) {
    m_RiskClient->SendAsync(msg);
}

// 接收风控结果
void OnRiskResult() {
    Message::PackMessage msg;
    if (m_RiskClient->PopFront(msg)) {
        if (msg.RiskReport.RiskStatus == Message::ECHECK_INIT) {
            // 风控通过，执行报单
            m_TradeGateWay->ReqInsertOrder(msg.OrderRequest);
        } else {
            // 风控拒绝，返回错误
            RejectOrder(msg, "Risk check failed");
        }
    }
}
```

## 数据流向设计

### 1. 策略报单流程 (带风控)

```
┌──────────┐
│  XQuant  │ 1. 发送报单请求
└────┬─────┘
     ↓ 共享内存 (1-3μs)
┌─────────────┐
│ OrderServer │ 2. SPSC 队列接收
└─────┬───────┘
      ↓
┌──────────────┐
│ WorkThread   │ 3. 读取请求队列
└──────┬───────┘
       ↓
  ┌────────────┐
  │ 是否需要风控? │
  └──┬─────┬───┘
     │Yes  │No
     ↓     ↓
┌──────────────┐    ┌──────────────┐
│ RiskServer   │    │ TradeGateWay │ 4a. 直接报单
│ (共享内存)    │    └──────────────┘
└──────┬───────┘
       ↓ 风控检查 (2-5μs)
┌──────────────┐
│ XRiskJudge   │ 5. 检查流速/撤单/自成交
└──────┬───────┘
       ↓
   ┌────────┐
   │通过?拒绝?│
   └──┬───┬──┘
      │   │
  Pass│   │Reject
      ↓   ↓
┌──────────────┐    ┌──────────────┐
│ TradeGateWay │    │ RejectOrder  │ 6a. 返回拒绝
└──────┬───────┘    └──────────────┘
       ↓
┌──────────────┐
│  柜台 API    │ 7. 发送到交易所
└──────┬───────┘
       ↓ OnRtnOrder / OnRtnTrade
┌──────────────┐
│ UpdateOrder  │ 8. 更新订单状态
└──────┬───────┘
       ↓
  ┌────────────────┐
  │ 双路径推送回报  │
  └──┬──────────┬──┘
     ↓          ↓
┌─────────┐ ┌──────────┐
│共享内存  │ │  网络    │ 9. 推送回报
│→ XQuant │ │→XWatcher │
└─────────┘ └──────────┘
```

**关键时间节点**:
- T0: XQuant 发送报单
- T1: OrderServer 接收 (+1-3μs)
- T2: 风控检查完成 (+2-5μs)
- T3: 发送到柜台 (+1-2μs)
- T4: 柜台回报 (+10-50μs, 取决于柜台)

**总延迟**: 14-60μs (优化后 < 20μs)

### 2. 手动报单流程 (无风控)

```
┌──────────┐
│ XMonitor │ 1. 手动报单
└────┬─────┘
     ↓ 网络
┌──────────────┐
│ HPPackClient │ 2. 接收网络消息
└──────┬───────┘
       ↓
┌──────────────┐
│MonitorThread │ 3. 处理请求
└──────┬───────┘
       ↓
┌──────────────┐
│ TradeGateWay │ 4. 直接报单 (不经过风控)
└──────┬───────┘
       ↓
┌──────────────┐
│  柜台 API    │ 5. 发送到交易所
└──────────────┘
```

**特点**:
- **不经过风控**: 手动报单由交易员负责
- **网络延迟**: 10-100ms (局域网) 或 50-200ms (广域网)
- **适用场景**: 紧急处理、特殊订单

### 3. 撤单流程

```
┌──────────┐
│策略/手动  │ 1. 发送撤单请求
└────┬─────┘
     ↓
┌──────────────┐
│  WorkThread  │ 2. 风控检查 (可选)
│MonitorThread │    - 检查撤单次数
└──────┬───────┘
       ↓
┌──────────────┐
│ TradeGateWay │ 3. 调用撤单 API
│ReqCancelOrder│
└──────┬───────┘
       ↓
┌──────────────┐
│  柜台 API    │ 4. 发送撤单到交易所
└──────┬───────┘
       ↓ OnRtnOrder (OrderStatus = Canceled)
┌──────────────┐
│ UpdateOrder  │ 5. 更新订单状态为已撤
└──────────────┘
```

## 线程模型设计

### 1. WorkThread (工作线程)

**职责**: 处理策略报单、风控交互、订单回报

**运行逻辑**:
```cpp
void* WorkThread(void* arg) {
    TraderEngine* engine = (TraderEngine*)arg;

    // 绑定 CPU
    Utils::ThreadBind(pthread_self(), engine->m_WorkThreadCPU);

    while (engine->m_Running) {
        // 1. 接收报单请求
        Message::PackMessage request;
        if (engine->m_RequestMessageQueue.Pop(request)) {

            // 2. 判断是否需要风控
            if (NeedRiskCheck(request)) {
                // 2a. 发送到风控
                engine->m_RiskClient->SendAsync(request);
            } else {
                // 2b. 直接报单
                engine->m_TradeGateWay->ReqInsertOrder(request.OrderRequest);
            }
        }

        // 3. 接收风控结果
        Message::PackMessage riskResult;
        if (engine->m_RiskClient->PopFront(riskResult)) {
            if (riskResult.RiskReport.RiskStatus == Message::ECHECK_INIT) {
                // 风控通过
                engine->m_TradeGateWay->ReqInsertOrder(riskResult.OrderRequest);
            } else {
                // 风控拒绝
                RejectOrder(riskResult);
            }
        }

        // 4. 处理订单回报
        Message::PackMessage report;
        if (engine->m_ReportMessageQueue.Pop(report)) {
            // 推送到共享内存
            engine->m_pOrderServer->Broadcast(report);
            // 推送到网络
            engine->m_HPPackClient->SendMessage(report);
        }

        // 5. 微秒级延迟
        usleep(1);
    }
}
```

**性能优化**:
- **CPU 绑定**: 独占 CPU 核心，避免上下文切换
- **无锁队列**: 减少锁竞争
- **批量处理**: 一次循环处理多条消息

### 2. MonitorThread (监控线程)

**职责**: 处理网络消息、状态上报

**运行逻辑**:
```cpp
void* MonitorThread(void* arg) {
    TraderEngine* engine = (TraderEngine*)arg;

    while (engine->m_Running) {
        // 1. 网络心跳
        if (Time - LastHeartBeatTime > 30s) {
            engine->m_HPPackClient->SendHeartBeat();
        }

        // 2. 定时查询
        if (Time - LastQueryTime > 60s) {
            engine->m_TradeGateWay->ReqQryPoistion();
            engine->m_TradeGateWay->ReqQryFund();
        }

        // 3. 状态上报
        if (Time - LastReportTime > 5s) {
            ReportStatus();
        }

        // 4. 秒级延迟 (非关键路径)
        sleep(1);
    }
}
```

**特点**:
- **非关键路径**: 不影响报单延迟
- **低频操作**: 秒级轮询
- **状态维护**: 维护连接、查询、上报

## 订单状态管理

### 订单状态机

```
      ┌──────────┐
      │  INIT    │ 初始化
      └────┬─────┘
           ↓ 发送报单
      ┌──────────┐
      │ PENDING  │ 已提交
      └────┬─────┘
           ↓ 柜台确认
    ┌──────┴──────┐
    ↓             ↓
┌────────┐   ┌──────────┐
│ACCEPTED│   │ REJECTED │ 拒绝
└────┬───┘   └──────────┘
     ↓
┌──────────┐
│PARTIAL   │ 部分成交
└────┬─────┘
     ↓
┌──────────┐
│ALL_TRADED│ 全部成交
└──────────┘

     ↓ 用户撤单
┌──────────┐
│CANCELED  │ 已撤销
└──────────┘
```

### 订单状态定义

```cpp
enum OrderStatus {
    EORDER_INIT        = 0,  // 初始化
    EORDER_PENDING     = 1,  // 已提交，等待柜台确认
    EORDER_ACCEPTED    = 2,  // 柜台已接受
    EORDER_PARTIAL     = 3,  // 部分成交
    EORDER_ALL_TRADED  = 4,  // 全部成交
    EORDER_CANCELED    = 5,  // 已撤销
    EORDER_REJECTED    = 6,  // 被拒绝
    EORDER_ERROR       = 99, // 错误
};
```

### 订单管理逻辑

```cpp
class OrderManager {
    // 订单映射
    std::unordered_map<int, Message::TOrderStatus> m_OrderMap;  // OrderToken -> OrderStatus

    // 订单更新
    void UpdateOrder(const Message::TOrderStatus& order) {
        auto it = m_OrderMap.find(order.OrderToken);
        if (it == m_OrderMap.end()) {
            // 新订单
            m_OrderMap[order.OrderToken] = order;
        } else {
            // 更新订单
            it->second = order;
        }

        // 推送回报
        BroadcastOrderUpdate(order);
    }

    // 成交更新
    void UpdateTrade(const Message::TOrderStatus& trade) {
        auto it = m_OrderMap.find(trade.OrderToken);
        if (it != m_OrderMap.end()) {
            // 更新成交数量
            it->second.TotalTradedVolume += trade.TradedVolume;
            it->second.TradedAvgPrice = CalculateAvgPrice(...);

            // 判断是否全部成交
            if (it->second.TotalTradedVolume >= it->second.SendVolume) {
                it->second.OrderStatus = EORDER_ALL_TRADED;
                it->second.DoneTime = Utils::getCurrentTimeUs();
            } else {
                it->second.OrderStatus = EORDER_PARTIAL;
            }
        }
    }
};
```

## 性能优化策略

### 1. CPU 绑定

```cpp
// 绑定工作线程到独立 CPU 核心
int cpuset = config.CPUSET[0];
Utils::ThreadBind(pthread_self(), cpuset);

// 绑定 OrderServer 到另一个核心
int orderServerCPU = config.CPUSET[1];
m_pOrderServer->Start(name, orderServerCPU);
```

**效果**:
- 避免 CPU 上下文切换
- 减少缓存失效
- 延迟降低 30-50%

### 2. 共享内存 IPC

```cpp
// 零拷贝通信
OrderServer* m_pOrderServer = new OrderServer();
m_pOrderServer->Start("OrderServer", cpuset);

// SPSC 无锁队列
while (true) {
    Message::PackMessage msg;
    if (m_pOrderServer->PopFront(msg)) {
        ProcessOrder(msg);
    }
}
```

**效果**:
- 通信延迟 1-3μs (vs 网络 100-1000μs)
- 无锁设计，无竞争

### 3. 无锁队列

```cpp
// 请求队列
LockFreeQueue<Message::PackMessage> m_RequestMessageQueue(10000);

// 回报队列
LockFreeQueue<Message::PackMessage> m_ReportMessageQueue(10000);

// 原子操作
bool Push(const T& item) {
    size_t h = head.load(std::memory_order_relaxed);
    size_t next = (h + 1) % capacity;
    if (next == tail.load(std::memory_order_acquire))
        return false;
    buffer[h] = item;
    head.store(next, std::memory_order_release);
    return true;
}
```

**效果**:
- 无锁竞争
- 延迟 < 100ns

### 4. 插件动态加载

```cpp
// 运行时加载插件
TradeGateWay* gateway = XPluginEngine<TradeGateWay>::LoadPlugin(soPath, errorString);
```

**优势**:
- 避免静态链接所有柜台 API
- 灵活切换柜台
- 减少二进制大小

### 5. 批量处理

```cpp
// 批量推送回报
std::vector<Message::PackMessage> batch;
while (m_ReportMessageQueue.Pop(msg)) {
    batch.push_back(msg);
    if (batch.size() >= 100) break;  // 批量上限
}

// 批量发送
for (auto& msg : batch) {
    m_HPPackClient->SendMessage(msg);
}
```

**效果**:
- 减少网络调用次数
- 提高吞吐量

## 可靠性保证

### 1. 订单持久化

```cpp
// 订单落盘
void PersistOrder(const Message::TOrderStatus& order) {
    std::ofstream ofs("orders.log", std::ios::app);
    ofs << Utils::getCurrentTimeUs() << ","
        << order.Account << ","
        << order.Ticker << ","
        << order.OrderRef << ","
        << order.OrderStatus << std::endl;
}
```

### 2. 异常恢复

```cpp
// 重连柜台
void Reconnect() {
    m_TradeGateWay->Release();
    sleep(5);
    m_TradeGateWay->CreateTraderAPI();
    m_TradeGateWay->ReqUserLogin();
}

// 恢复订单状态
void RecoverOrders() {
    m_TradeGateWay->ReqQryOrder();  // 查询所有订单
}
```

### 3. 错误处理

```cpp
// API 错误码映射
std::unordered_map<int, std::string> m_ErrorMap;

// 加载错误码
void LoadErrorCodes() {
    Utils::YAMLConfig config;
    config.LoadConfig("Config/CTPError.yml");
    for (auto& item : config["Errors"]) {
        m_ErrorMap[item["code"]] = item["message"];
    }
}

// 错误处理
void OnError(int errorCode) {
    std::string errorMsg = m_ErrorMap[errorCode];
    Logger::Error("API Error: {} - {}", errorCode, errorMsg);

    // 根据错误类型处理
    if (errorCode == -3) {
        // 频率限制，延迟重试
        sleep(1);
    } else if (errorCode == -7) {
        // 未登录，重新登录
        Reconnect();
    }
}
```

## 总结

XTrader 通过以下设计实现高性能交易执行：

1. **插件架构**: 统一接口对接多柜台，动态加载
2. **双线程模型**: WorkThread 处理关键路径，MonitorThread 处理非关键操作
3. **多路径通信**: 共享内存 (策略) + 网络 (手动)
4. **风控集成**: 与 XRiskJudge 深度集成，订单前置检查
5. **状态管理**: 精确的订单状态机，支持持久化与恢复
6. **性能优化**: CPU 绑定、无锁队列、批量处理

**关键优势**:
- Tick2Order < 20μs
- 订单不丢失
- 多柜台支持
- 风控集成
- 易扩展

**下一步**: 查看技术实现详解了解具体代码实现。

---
作者: @scorpiostudio @yutiansut
