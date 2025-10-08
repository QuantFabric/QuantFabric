# XMarketCenter 设计理念与架构

## 设计理念

### 1. 核心目标

XMarketCenter 作为量化交易系统的**行情网关**，其设计理念围绕以下核心目标：

#### 1.1 极致低延迟

**设计原则**:
- **零拷贝数据传输**: 行情数据从柜台 API 到策略引擎全程避免不必要的内存拷贝
- **无锁编程**: 关键路径使用无锁数据结构，消除互斥锁开销
- **CPU 亲和性**: 关键线程绑定到隔离 CPU，避免上下文切换
- **异步处理**: 柜台 API 回调与数据分发异步进行，避免阻塞

**实现机制**:
```cpp
// 行情接收路径（关键路径）
OnRtnDepthMarketData()  // 柜台回调（5-10μs）
    ↓
数据转换 (CTP → TFutureMarketData)  // 内存拷贝（2-3μs）
    ↓
m_MarketMessageQueue.Push()  // 无锁队列写入（1-2μs）
    ↓
主线程 Pop() 读取  // 无锁队列读取（1-2μs）
    ↓
共享内存发布 Publish()  // 零拷贝（2-3μs）
```

**延迟目标**: 柜台回调 → 策略接收 < 15μs

#### 1.2 高可靠性

**设计原则**:
- **自动重连**: 柜台断线后自动重连，无需人工干预
- **心跳保活**: 定期发送心跳，检测连接状态
- **异常隔离**: 插件异常不影响主进程稳定性
- **日志审计**: 完整记录行情接收、数据分发、异常事件

**容错机制**:
```cpp
// 自动重连逻辑
void OnFrontDisconnected(int reason) {
    m_ConnectedStatus = DISCONNECTED;
    FMTLOG(fmtlog::WRN, "Front disconnected, reason={}", reason);

    // 延迟重连，避免频繁连接
    sleep(5);

    // 重新登录
    ReqUserLogin();

    // 重新订阅
    SubscribeMarketData();
}

// 心跳检测
void HeartBeat() {
    if (time_now - m_LastRecvTime > TIMEOUT) {
        FMTLOG(fmtlog::ERR, "Market data timeout, reconnecting...");
        Reconnect();
    }
}
```

#### 1.3 高扩展性

**设计原则**:
- **插件架构**: 通过动态加载支持多种柜台 API
- **接口统一**: 所有柜台插件实现统一的 `MarketGateWay` 接口
- **配置驱动**: 通过配置文件切换柜台，无需重新编译

**插件模式优势**:
1. **解耦柜台依赖**: 主程序不依赖特定柜台 API
2. **灵活切换**: 运行时指定不同插件 .so
3. **隔离故障**: 插件异常不影响主框架
4. **并行开发**: 不同团队独立开发插件

### 2. 架构分层

```
┌─────────────────────────────────────────┐
│           应用层 (XMarketCenter)         │
│  - 配置加载                              │
│  - 插件管理                              │
│  - 网络通信 (HPPackClient)               │
├─────────────────────────────────────────┤
│          插件层 (MarketGateWay)          │
│  - CTPMarketGateWay                     │
│  - REMMarketGateWay                     │
│  - YDMarketGateWay                      │
├─────────────────────────────────────────┤
│           通信层 (IPC)                   │
│  - 无锁队列 (LockFreeQueue)              │
│  - 共享内存 (SHMServer)                  │
│  - 网络通信 (HPSocket)                   │
├─────────────────────────────────────────┤
│          柜台层 (Broker API)             │
│  - CTP API                              │
│  - REM API                              │
│  - YD API                               │
└─────────────────────────────────────────┘
```

### 3. 数据流设计

#### 3.1 行情接收流程

```
交易所
  ↓ 网络
柜台前置服务器
  ↓ 柜台API回调
MarketGateWay 插件
  ├─> OnRtnDepthMarketData() 回调
  ├─> 数据格式转换
  └─> m_MarketMessageQueue.Push()
       ↓
XMarketCenter 主线程
  ├─> m_MarketMessageQueue.Pop()
  ├─> 解析消息类型
  └─> 分发数据
       ├─> 共享内存 (MarketServer) → XQuant/HFTrader
       └─> 网络 (HPPackClient) → XWatcher → XMonitor
```

**关键设计点**:

1. **异步解耦**: 柜台回调线程只负责数据写入队列，主线程负责分发
2. **无锁传输**: 使用 `LockFreeQueue` 避免线程间竞争
3. **双路分发**: 同时支持共享内存（低延迟策略）和网络（监控展示）

#### 3.2 数据转换原理

**CTP → TFutureMarketData 转换**:

```cpp
void ConvertCTPData(CThostFtdcDepthMarketDataField* pData,
                    MarketData::TFutureMarketData& out) {
    // 合约信息
    strncpy(out.Ticker, pData->InstrumentID, sizeof(out.Ticker));
    strncpy(out.ExchangeID, pData->ExchangeID, sizeof(out.ExchangeID));

    // 时间信息
    strncpy(out.UpdateTime, pData->UpdateTime, sizeof(out.UpdateTime));
    out.MillSec = pData->UpdateMillisec;

    // 价格信息
    out.LastPrice = pData->LastPrice;
    out.OpenPrice = pData->OpenPrice;
    out.HighestPrice = pData->HighestPrice;
    out.LowestPrice = pData->LowestPrice;

    // 五档行情
    out.BidPrice1 = pData->BidPrice1;
    out.BidVolume1 = pData->BidVolume1;
    out.AskPrice1 = pData->AskPrice1;
    out.AskVolume1 = pData->AskVolume1;
    // ... 其他档位

    // 成交信息
    out.Volume = pData->Volume;
    out.Turnover = pData->Turnover;
    out.OpenInterest = pData->OpenInterest;

    // 时间戳
    out.RecvMarketTime = Utils::getTimeUs();
}
```

**设计要点**:
- 字段映射统一化，便于不同柜台数据对齐
- 添加接收时间戳，用于延迟统计
- 使用 `strncpy` 防止缓冲区溢出

### 4. 性能优化策略

#### 4.1 内存优化

**预分配策略**:
```cpp
// 无锁队列预分配
LockFreeQueue<Message::PackMessage> m_MarketMessageQueue(1 << 14);  // 16K 元素

// 避免动态分配
struct MarketData {
    char Ticker[32];     // 固定大小
    char ExchangeID[16]; // 避免 std::string
    // ... 其他字段
};
```

**缓存行对齐**:
```cpp
// 避免伪共享
struct alignas(64) MarketData {  // 64 字节对齐（CPU 缓存行大小）
    // ... 字段定义
};
```

#### 4.2 CPU 优化

**关键线程绑定**:
```cpp
// 主线程绑定到 CPU 5
Utils::BindCPU(m_MarketCenterConfig.CPUSET);

// 柜台 API 回调线程由 API 库创建，无法直接绑定
// 但通过进程 CPU 亲和性限制：taskset -c 5 ./XMarketCenter
```

**编译优化**:
```cmake
set(CMAKE_CXX_FLAGS "-O3 -march=native -mtune=native")
```

#### 4.3 网络优化

**Onload 内核旁路**:
```bash
# 使用 Solarflare Onload 加速
onload --profile=latency ./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so
```

**UDP 多播行情** (REM):
```cpp
// REM 支持 UDP 多播，延迟更低
REMMarketSourceConfig:
  MultiCastIP: 239.0.0.1
  MultiCastPort: 6666
  LocalIP: 192.168.1.10
  LocalPort: 7777
```

### 5. 设计模式应用

#### 5.1 插件模式 (Plugin Pattern)

**意图**: 动态加载不同柜台行情插件

**实现**:
```cpp
// 插件接口
class MarketGateWay {
public:
    virtual void Run() = 0;
    virtual bool LoadAPIConfig() = 0;
    // ... 其他虚函数
};

// 插件加载
template<typename T>
class XPluginEngine {
public:
    static T* LoadPlugin(const std::string& soPath, std::string& errorString) {
        void* handle = dlopen(soPath.c_str(), RTLD_LAZY);
        if (!handle) {
            errorString = dlerror();
            return nullptr;
        }

        typedef T* (*CreateFunc)();
        CreateFunc createFunc = (CreateFunc)dlsym(handle, "CreatePlugin");
        if (!createFunc) {
            errorString = "Symbol 'CreatePlugin' not found";
            dlclose(handle);
            return nullptr;
        }

        return createFunc();
    }
};

// 插件使用
MarketGateWay* gateway = XPluginEngine<MarketGateWay>::LoadPlugin(
    "./libCTPMarketGateWay.so", errorString);
gateway->Run();
```

#### 5.2 生产者-消费者模式 (Producer-Consumer)

**意图**: 柜台回调线程生产，主线程消费

**实现**:
```cpp
// 生产者（柜台回调线程）
void OnRtnDepthMarketData(CThostFtdcDepthMarketDataField* pData) {
    Message::PackMessage msg;
    msg.MessageType = Message::EFutureMarket;
    ConvertCTPData(pData, msg.FutureMarketData);

    m_MarketMessageQueue.Push(msg);  // 无锁写入
}

// 消费者（主线程）
void MainLoop() {
    while (true) {
        Message::PackMessage msg;
        if (m_MarketMessageQueue.Pop(msg)) {
            // 处理消息
            DispatchMarketData(msg);
        }
    }
}
```

#### 5.3 发布-订阅模式 (Publish-Subscribe)

**意图**: 行情数据一对多分发

**实现**:
```cpp
// 发布者
class XMarketCenter {
    void PublishMarketData(const MarketData::TFutureMarketData& data) {
        // 发布到共享内存
        m_PubServer->Publish("MarketServer", &data, sizeof(data));

        // 发布到网络
        Message::PackMessage msg;
        msg.MessageType = Message::EFutureMarket;
        msg.FutureMarketData = data;
        m_PackClient->SendData((unsigned char*)&msg, sizeof(msg));
    }
};

// 订阅者
class XQuant {
    void SubscribeMarketData() {
        m_MarketConn.Start("MarketServer");  // 订阅共享内存

        while (true) {
            MarketData::TFutureMarketData data;
            if (m_MarketConn.Receive(&data, sizeof(data))) {
                OnMarketData(data);  // 回调处理
            }
        }
    }
};
```

### 6. 可靠性保证

#### 6.1 断线重连机制

**策略**:
1. **指数退避**: 重连间隔逐步增加，避免频繁连接
2. **状态管理**: 严格管理连接状态转换
3. **资源清理**: 重连前释放旧资源

**实现**:
```cpp
class CTPMarketGateWay {
private:
    int m_ReconnectCount = 0;
    const int MAX_RECONNECT_INTERVAL = 60;  // 最大重连间隔 60 秒

    void OnFrontDisconnected(int reason) {
        m_ConnectedStatus = DISCONNECTED;

        // 计算退避间隔
        int interval = std::min(m_ReconnectCount * 5, MAX_RECONNECT_INTERVAL);
        FMTLOG(fmtlog::WRN, "Disconnected, reconnect after {} seconds", interval);

        sleep(interval);

        // 重新初始化 API
        if (m_pMdUserApi) {
            m_pMdUserApi->Release();
            m_pMdUserApi = nullptr;
        }

        CreateAPI();
        m_pMdUserApi->RegisterSpi(this);
        m_pMdUserApi->RegisterFront(m_FrontAddr);
        m_pMdUserApi->Init();

        m_ReconnectCount++;
    }

    void OnRspUserLogin(CThostFtdcRspUserLoginField* pRspUserLogin,
                        CThostFtdcRspInfoField* pRspInfo, int nRequestID, bool bIsLast) {
        if (pRspInfo && pRspInfo->ErrorID == 0) {
            m_ConnectedStatus = CONNECTED;
            m_ReconnectCount = 0;  // 重置重连计数
            FMTLOG(fmtlog::INF, "Login success");

            // 重新订阅
            SubscribeMarketData();
        }
    }
};
```

#### 6.2 数据完整性

**序列号机制**:
```cpp
struct TFutureMarketData {
    unsigned int Tick;  // 自增序列号
    // ... 其他字段
};

// 发送端
void SendMarketData(TFutureMarketData& data) {
    data.Tick = m_TickCount++;  // 自增
    Publish(data);
}

// 接收端检测丢包
void OnMarketData(const TFutureMarketData& data) {
    if (data.Tick != m_LastTick + 1) {
        FMTLOG(fmtlog::WRN, "Market data loss detected: {} -> {}",
               m_LastTick, data.Tick);
    }
    m_LastTick = data.Tick;
}
```

### 7. 监控与诊断

#### 7.1 性能监控

**延迟统计**:
```cpp
struct LatencyStats {
    unsigned long min = ULONG_MAX;
    unsigned long max = 0;
    unsigned long sum = 0;
    unsigned long count = 0;

    void Update(unsigned long latency) {
        min = std::min(min, latency);
        max = std::max(max, latency);
        sum += latency;
        count++;
    }

    double GetAverage() { return count > 0 ? (double)sum / count : 0; }
};

// 使用
LatencyStats stats;
unsigned long recvTime = Utils::getTimeUs();
unsigned long latency = recvTime - data.ExchangeTime;  // 需要柜台提供交易所时间
stats.Update(latency);

// 定期输出
if (stats.count % 10000 == 0) {
    FMTLOG(fmtlog::INF, "Latency stats: min={}, max={}, avg={}",
           stats.min, stats.max, stats.GetAverage());
}
```

#### 7.2 事件日志

**关键事件记录**:
```cpp
// 连接事件
FMTLOG(fmtlog::INF, "Front connected: {}", m_FrontAddr);

// 登录事件
FMTLOG(fmtlog::INF, "Login success, TradingDay={}", pRspUserLogin->TradingDay);

// 订阅事件
FMTLOG(fmtlog::INF, "Subscribe {} tickers", m_TickerList.size());

// 行情接收
FMTLOG(fmtlog::DBG, "Recv market: {} LastPrice={}", data.Ticker, data.LastPrice);

// 异常事件
FMTLOG(fmtlog::ERR, "OnRspError: ErrorID={}, ErrorMsg={}",
       pRspInfo->ErrorID, pRspInfo->ErrorMsg);
```

### 8. 设计权衡

#### 8.1 内存 vs 性能

**选择**: 牺牲少量内存，换取性能提升

- 预分配大容量队列 (16K 元素)
- 固定大小字段，避免动态分配
- 缓存行对齐，避免伪共享

#### 8.2 灵活性 vs 性能

**选择**: 通过编译期优化保证性能，运行时保证灵活性

- 插件架构：运行时灵活切换柜台
- 模板优化：编译期展开，零开销抽象
- 配置驱动：运行时调整参数

#### 8.3 可靠性 vs 延迟

**选择**: 在关键路径保证低延迟，在非关键路径保证可靠性

- 行情接收：无锁队列，不检查失败（假设足够容量）
- 网络发送：异步发送，不阻塞行情接收
- 重连机制：单独线程处理，不影响已建立连接的数据接收

## 总结

XMarketCenter 的设计理念体现了**高性能量化交易系统**的核心思想：

1. **极致性能**: 微秒级延迟目标驱动所有设计决策
2. **高可靠性**: 自动容错，无人值守运行
3. **强扩展性**: 插件架构，支持多种柜台
4. **易维护性**: 配置驱动，日志完善

通过**分层架构、设计模式、性能优化**的综合运用，XMarketCenter 成为连接交易所与策略引擎的高效桥梁。

**下一步**: 阅读 [技术实现详解](02_TechnicalImplementation.md) 了解具体实现细节。
