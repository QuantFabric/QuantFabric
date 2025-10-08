# XMarketCenter 技术实现详解

## 插件加载机制

### 1. 插件接口定义

**MarketGateWay.hpp** - 插件基类：

```cpp
class MarketGateWay {
public:
    // ========== 纯虚函数（插件必须实现） ==========
    virtual bool LoadAPIConfig() = 0;           // 加载柜台配置
    virtual void Run() = 0;                     // 启动行情接收
    virtual void GetCommitID(std::string& CommitID,
                             std::string& UtilsCommitID) = 0;
    virtual void GetAPIVersion(std::string& APIVersion) = 0;

    // ========== 公共接口（基类提供） ==========
    void SetLogger(Utils::Logger* logger) {
        m_Logger = logger;
    }

    void SetMarketConfig(const Utils::MarketCenterConfig& config) {
        m_MarketCenterConfig = config;
        LoadAPIConfig();
        LoadTickerList();
    }

protected:
    // ========== 受保护成员（插件可访问） ==========
    Utils::MarketCenterConfig m_MarketCenterConfig;
    Utils::Logger* m_Logger;

    // 行情消息队列（插件写入，主程序读取）
    LockFreeQueue<Message::PackMessage> m_MarketMessageQueue;

    // Ticker 映射表
    std::unordered_map<std::string, std::string> m_TickerExchangeMap;
    std::vector<Utils::TickerProperty> m_TickerPropertyList;

    // 辅助函数
    void PrintMarketData(const MarketData::TFutureMarketData& data);
};
```

**设计要点**:
1. **接口最小化**: 只要求实现 4 个纯虚函数
2. **配置注入**: 通过 `SetMarketConfig()` 注入配置
3. **队列共享**: 基类提供 `m_MarketMessageQueue`，插件写入，主程序读取
4. **日志统一**: 基类管理 Logger，插件直接使用

### 2. 插件实现示例

**CTPMarketGateWay.cpp** - CTP 插件实现：

```cpp
class CTPMarketGateWay : public MarketGateWay,
                         public CThostFtdcMdSpi {  // 多重继承
public:
    bool LoadAPIConfig() override {
        std::string errorString;
        if (!Utils::LoadCTPMarkeSourceConfig(
                m_MarketCenterConfig.APIConfig.c_str(),
                m_CTPConfig, errorString)) {
            m_Logger->Log->error("LoadAPIConfig failed: {}", errorString);
            return false;
        }

        strncpy(m_FrontAddr, m_CTPConfig.FrontAddr.c_str(), sizeof(m_FrontAddr));
        strncpy(m_BrokerID, m_CTPConfig.BrokerID.c_str(), sizeof(m_BrokerID));
        strncpy(m_Account, m_CTPConfig.Account.c_str(), sizeof(m_Account));
        strncpy(m_Password, m_CTPConfig.Password.c_str(), sizeof(m_Password));

        return true;
    }

    void Run() override {
        // 创建 CTP API 实例
        m_pMdUserApi = CThostFtdcMdApi::CreateFtdcMdApi();
        m_pMdUserApi->RegisterSpi(this);
        m_pMdUserApi->RegisterFront(m_FrontAddr);

        m_Logger->Log->info("CTP Market Gateway starting, Front={}", m_FrontAddr);

        // 启动 API（阻塞调用，内部启动线程）
        m_pMdUserApi->Init();

        // Join 等待（实际由 CTP 内部线程处理回调）
        m_pMdUserApi->Join();
    }

    // ========== CTP 回调函数 ==========
    void OnFrontConnected() override {
        m_ConnectedStatus = CONNECTED;
        m_Logger->Log->info("OnFrontConnected");

        // 发送登录请求
        CThostFtdcReqUserLoginField req = {};
        strncpy(req.BrokerID, m_BrokerID, sizeof(req.BrokerID));
        strncpy(req.UserID, m_Account, sizeof(req.UserID));
        strncpy(req.Password, m_Password, sizeof(req.Password));

        m_pMdUserApi->ReqUserLogin(&req, ++m_RequestID);
    }

    void OnRspUserLogin(CThostFtdcRspUserLoginField* pRspUserLogin,
                        CThostFtdcRspInfoField* pRspInfo,
                        int nRequestID, bool bIsLast) override {
        if (pRspInfo && pRspInfo->ErrorID != 0) {
            m_Logger->Log->error("Login failed: ErrorID={}, ErrorMsg={}",
                                 pRspInfo->ErrorID, pRspInfo->ErrorMsg);
            return;
        }

        m_Logger->Log->info("Login success, TradingDay={}", pRspUserLogin->TradingDay);

        // 订阅行情
        SubscribeMarketData();
    }

    void OnRtnDepthMarketData(CThostFtdcDepthMarketDataField* pDepthMarketData) override {
        // 过滤不需要的 Ticker
        if (m_TickerExchangeMap.find(pDepthMarketData->InstrumentID) ==
            m_TickerExchangeMap.end()) {
            return;
        }

        // 构造消息
        Message::PackMessage message;
        memset(&message, 0, sizeof(message));
        message.MessageType = Message::EMessageType::EFutureMarket;

        // 数据转换
        ConvertCTPMarketData(pDepthMarketData, message.FutureMarketData);

        // 写入队列（无锁）
        m_MarketMessageQueue.Push(message);

        // 调试日志（生产环境关闭）
        if (m_Logger->Log->should_log(spdlog::level::debug)) {
            PrintMarketData(message.FutureMarketData);
        }
    }

private:
    void SubscribeMarketData() {
        int count = m_TickerPropertyList.size();
        char** tickers = new char*[count];

        for (int i = 0; i < count; i++) {
            tickers[i] = (char*)m_TickerPropertyList[i].Ticker.c_str();
        }

        m_pMdUserApi->SubscribeMarketData(tickers, count);
        m_Logger->Log->info("Subscribed {} tickers", count);

        delete[] tickers;
    }

    void ConvertCTPMarketData(CThostFtdcDepthMarketDataField* pData,
                              MarketData::TFutureMarketData& out) {
        out.Tick = m_TickCount++;

        strncpy(out.Ticker, pData->InstrumentID, sizeof(out.Ticker));
        strncpy(out.ExchangeID, pData->ExchangeID, sizeof(out.ExchangeID));
        strncpy(out.UpdateTime, pData->UpdateTime, sizeof(out.UpdateTime));
        out.MillSec = pData->UpdateMillisec;

        out.LastPrice = pData->LastPrice;
        out.Volume = pData->Volume;
        out.Turnover = pData->Turnover;
        out.OpenInterest = pData->OpenInterest;

        out.BidPrice1 = pData->BidPrice1;
        out.BidVolume1 = pData->BidVolume1;
        out.AskPrice1 = pData->AskPrice1;
        out.AskVolume1 = pData->AskVolume1;

        // 记录接收时间
        out.RecvMarketTime = Utils::getTimeUs();
    }

    CThostFtdcMdApi* m_pMdUserApi = nullptr;
    Utils::CTPMarketSourceConfig m_CTPConfig;
    char m_FrontAddr[256];
    char m_BrokerID[16];
    char m_Account[16];
    char m_Password[16];
    int m_RequestID = 0;
    unsigned int m_TickCount = 0;
    int m_ConnectedStatus = 0;
};

// ========== 插件导出函数 ==========
extern "C" {
    MarketGateWay* CreatePlugin() {
        return new CTPMarketGateWay();
    }
}
```

### 3. 插件加载流程

**XPluginEngine.hpp** - 插件加载引擎：

```cpp
template<typename T>
class XPluginEngine {
public:
    static T* LoadPlugin(const std::string& soPath, std::string& errorString) {
        // 1. 加载动态库
        void* handle = dlopen(soPath.c_str(), RTLD_LAZY);
        if (!handle) {
            errorString = dlerror();
            return nullptr;
        }

        // 2. 查找创建函数符号
        typedef T* (*CreateFunc)();
        CreateFunc createFunc = (CreateFunc)dlsym(handle, "CreatePlugin");
        if (!createFunc) {
            errorString = "Symbol 'CreatePlugin' not found";
            dlclose(handle);
            return nullptr;
        }

        // 3. 调用创建函数
        T* plugin = createFunc();
        if (!plugin) {
            errorString = "CreatePlugin() returned nullptr";
            dlclose(handle);
            return nullptr;
        }

        // 注意：不调用 dlclose()，保持插件加载
        return plugin;
    }
};
```

**XMarketCenter.cpp** - 主程序加载插件：

```cpp
void XMarketCenter::LoadMarketGateWay(const std::string& soPath) {
    std::string errorString;

    // 加载插件
    m_MarketGateWay = XPluginEngine<MarketGateWay>::LoadPlugin(soPath, errorString);

    if (nullptr == m_MarketGateWay) {
        FMTLOG(fmtlog::ERR, "LoadMarketGateWay {} failed: {}", soPath, errorString);
        exit(-1);
    }

    FMTLOG(fmtlog::INF, "LoadMarketGateWay {} success", soPath);

    // 注入依赖
    m_MarketGateWay->SetLogger(Utils::Singleton<Utils::Logger>::GetInstance());
    m_MarketGateWay->SetMarketConfig(m_MarketCenterConfig);

    // 获取插件版本信息
    std::string commitID, utilsCommitID, apiVersion;
    m_MarketGateWay->GetCommitID(commitID, utilsCommitID);
    m_MarketGateWay->GetAPIVersion(apiVersion);

    FMTLOG(fmtlog::INF, "Plugin info: CommitID={}, UtilsCommitID={}, APIVersion={}",
           commitID, utilsCommitID, apiVersion);
}
```

## 无锁队列原理

### 1. LockFreeQueue 实现

**核心数据结构**:

```cpp
template<typename T>
class LockFreeQueue {
public:
    explicit LockFreeQueue(size_t capacity)
        : m_Capacity(capacity),
          m_Head(0),
          m_Tail(0) {
        m_Buffer = new T[capacity];
    }

    ~LockFreeQueue() {
        delete[] m_Buffer;
    }

    // 生产者（单线程）
    bool Push(const T& item) {
        size_t head = m_Head.load(std::memory_order_relaxed);
        size_t next = (head + 1) % m_Capacity;

        // 检查队列是否已满
        if (next == m_Tail.load(std::memory_order_acquire)) {
            return false;  // 队列满
        }

        m_Buffer[head] = item;
        m_Head.store(next, std::memory_order_release);
        return true;
    }

    // 消费者（单线程）
    bool Pop(T& item) {
        size_t tail = m_Tail.load(std::memory_order_relaxed);

        // 检查队列是否为空
        if (tail == m_Head.load(std::memory_order_acquire)) {
            return false;  // 队列空
        }

        item = m_Buffer[tail];
        m_Tail.store((tail + 1) % m_Capacity, std::memory_order_release);
        return true;
    }

private:
    T* m_Buffer;
    size_t m_Capacity;
    std::atomic<size_t> m_Head;  // 写指针
    std::atomic<size_t> m_Tail;  // 读指针
};
```

**关键技术**:

1. **环形缓冲区**: 使用取模实现循环使用
2. **原子操作**: `std::atomic` 保证内存可见性
3. **内存序**:
   - `relaxed`: 无同步，仅保证原子性
   - `acquire`: 读操作，保证之后的读写不会重排到之前
   - `release`: 写操作，保证之前的读写不会重排到之后
4. **SPSC 假设**: 单生产者单消费者，避免 CAS 开销

### 2. 内存序详解

**为什么需要内存序**？

```cpp
// 错误示例：没有内存屏障
// 线程 A（生产者）
m_Buffer[head] = item;      // 1. 写数据
m_Head = next;              // 2. 更新 head

// 线程 B（消费者）
if (tail != m_Head) {       // 3. 读 head
    item = m_Buffer[tail];  // 4. 读数据
}
```

**问题**: CPU 可能重排指令，导致线程 B 在线程 A 写入数据之前就读取 head，读到脏数据。

**正确示例：使用内存序**:

```cpp
// 线程 A（生产者）
m_Buffer[head] = item;
m_Head.store(next, std::memory_order_release);  // release 保证之前的写不会重排到之后

// 线程 B（消费者）
if (tail == m_Head.load(std::memory_order_acquire)) {  // acquire 保证之后的读不会重排到之前
    return false;
}
item = m_Buffer[tail];
```

**内存序保证**:
- `release` 写保证：之前的所有读写操作都在 release 之前完成
- `acquire` 读保证：之后的所有读写操作都在 acquire 之后完成
- 组合效果：生产者写入数据 → release 写 head → acquire 读 head → 消费者读取数据（顺序保证）

### 3. 性能对比

| 队列类型 | 延迟 | 吞吐量 | 适用场景 |
|---------|------|--------|---------|
| `std::queue` + `std::mutex` | 200-500ns | 低 | 通用场景 |
| `LockFreeQueue` (SPSC) | 20-50ns | 高 | 生产者-消费者 |
| `moodycamel::ConcurrentQueue` (MPMC) | 50-100ns | 中 | 多生产者多消费者 |

**XMarketCenter 选择 SPSC 的原因**:
- 柜台回调线程是单一生产者
- 主线程是单一消费者
- 性能最优

## 共享内存发布

### 1. SHMServer 架构

**发布-订阅模型**:

```
XMarketCenter (Publisher)
       ↓
  SHMServer (Broker)
       ↓
  ┌─────┴─────┬─────┬─────┐
  ↓           ↓     ↓     ↓
XQuant1   XQuant2  ...  XQuantN (Subscribers)
```

**内存布局**:

```
共享内存段: "MarketServer"
┌──────────────────────────────┐
│  Control Block (元数据)        │
│  - 订阅者数量                  │
│  - 写指针                     │
├──────────────────────────────┤
│  Channel 1 (XQuant1)          │
│  ├─ SPSC Queue (Server→Client)│
│  └─ SPSC Queue (Client→Server)│
├──────────────────────────────┤
│  Channel 2 (XQuant2)          │
│  ├─ SPSC Queue (Server→Client)│
│  └─ SPSC Queue (Client→Server)│
├──────────────────────────────┤
│  ...                          │
└──────────────────────────────┘
```

### 2. 发布流程

**XMarketCenter 发布代码**:

```cpp
class XMarketCenter {
private:
    PubServer* m_PubServer;  // 发布服务器

public:
    void Run() {
        // 1. 创建发布服务器
        m_PubServer = new PubServer();
        m_PubServer->Start(m_MarketCenterConfig.MarketServer);

        // 2. 主循环
        while (true) {
            Message::PackMessage msg;
            if (m_MarketMessageQueue.Pop(msg)) {  // 从无锁队列读取
                // 3. 发布到共享内存
                if (msg.MessageType == Message::EFutureMarket) {
                    m_PubServer->Publish(&msg.FutureMarketData,
                                         sizeof(msg.FutureMarketData));
                }

                // 4. 发送到网络（监控）
                m_PackClient->SendData((unsigned char*)&msg, sizeof(msg));
            }
        }
    }
};
```

**PubServer 实现**:

```cpp
class PubServer {
public:
    void Start(const std::string& name) {
        // 创建共享内存
        m_SHMKey = ftok(name.c_str(), 0);
        m_SHMID = shmget(m_SHMKey, SHM_SIZE, IPC_CREAT | 0666);

        m_SHMAddr = (char*)shmat(m_SHMID, nullptr, 0);

        // 初始化控制块
        ControlBlock* ctrl = (ControlBlock*)m_SHMAddr;
        ctrl->subscriberCount = 0;
        ctrl->writeIndex = 0;
    }

    void Publish(const void* data, size_t len) {
        ControlBlock* ctrl = (ControlBlock*)m_SHMAddr;

        // 遍历所有订阅者的发送队列
        for (int i = 0; i < ctrl->subscriberCount; i++) {
            Channel* channel = GetChannel(i);
            SPSCQueue* queue = &channel->sendQueue;  // Server→Client 队列

            // 写入队列（无锁）
            queue->Push(data, len);
        }
    }

private:
    key_t m_SHMKey;
    int m_SHMID;
    char* m_SHMAddr;
};
```

### 3. 订阅流程

**XQuant 订阅代码**:

```cpp
class XQuant {
private:
    SHMIPC::SHMConnection<MarketData::TFutureMarketData, ClientConf> m_MarketConn;

public:
    void Run() {
        // 1. 连接共享内存
        m_MarketConn.Start("MarketServer");

        // 2. 接收循环
        while (true) {
            MarketData::TFutureMarketData data;
            if (m_MarketConn.Receive(&data, sizeof(data))) {
                // 3. 处理行情
                OnMarketData(data);
            }
        }
    }

    void OnMarketData(const MarketData::TFutureMarketData& data) {
        // 策略逻辑
        if (ShouldBuy(data)) {
            SendOrder(...);
        }
    }
};
```

## 网络通信机制

### 1. HPSocket Pack 协议

**协议格式**:

```
┌────────────┬────────────────────┐
│ 4 字节长度 │  N 字节数据         │
│  (Header)  │   (Body)           │
└────────────┴────────────────────┘

长度字段：包含 Header + Body 的总长度（大端序）
数据字段：Message::PackMessage 结构体
```

**发送流程**:

```cpp
class HPPackClient {
public:
    bool SendData(unsigned char* buffer, int len) {
        // 1. 构造 Pack 包
        int totalLen = sizeof(int) + len;
        unsigned char* packet = new unsigned char[totalLen];

        // 2. 写入长度（大端序）
        *(int*)packet = htonl(totalLen);

        // 3. 写入数据
        memcpy(packet + sizeof(int), buffer, len);

        // 4. 发送
        bool ret = HP_Client_Send(m_pClient, packet, totalLen);

        delete[] packet;
        return ret;
    }

private:
    HP_TcpPackClient m_pClient;  // HPSocket Pack 客户端
};
```

**接收回调**:

```cpp
EnHandleResult OnReceive(HP_TcpPackClient pSender, CONNID dwConnID,
                         const BYTE* pData, int iLength) {
    // pData 已经是解包后的数据（HPSocket 自动处理）
    Message::PackMessage* msg = (Message::PackMessage*)pData;

    switch (msg->MessageType) {
        case Message::EFutureMarket:
            // 处理行情
            break;
        case Message::EOrderStatus:
            // 处理订单回报
            break;
        // ... 其他消息类型
    }

    return HR_OK;
}
```

### 2. 消息分发

**消息类型定义**:

```cpp
namespace Message {
    enum EMessageType {
        EEventLog = 1,
        EFutureMarket = 2,
        EStockMarket = 3,
        EOrderStatus = 4,
        EAccountPosition = 5,
        EAccountFund = 6,
        ERiskReport = 7,
        EColoStatus = 8,
        EAppStatus = 9,
        // ...
    };

    struct PackMessage {
        EMessageType MessageType;
        union {
            TFutureMarketData FutureMarketData;
            TStockMarketData StockMarketData;
            TOrderStatus OrderStatus;
            TAccountPosition AccountPosition;
            TAccountFund AccountFund;
            TRiskReport RiskReport;
            // ...
        };
    };
}
```

**XMarketCenter 发送**:

```cpp
void XMarketCenter::DispatchMessage(Message::PackMessage& msg) {
    // 1. 发送到共享内存（策略）
    if (m_MarketCenterConfig.ToMonitor) {
        if (msg.MessageType == Message::EFutureMarket) {
            m_PubServer->Publish(&msg.FutureMarketData,
                                 sizeof(msg.FutureMarketData));
        }
    }

    // 2. 发送到网络（监控）
    m_PackClient->SendData((unsigned char*)&msg, sizeof(msg));
}
```

**XWatcher 转发**:

```cpp
EnHandleResult XWatcher::OnReceive(HP_TcpPackServer pSender, CONNID dwConnID,
                                   const BYTE* pData, int iLength) {
    Message::PackMessage* msg = (Message::PackMessage*)pData;

    // 转发到 XServer
    m_ServerClient->SendData((unsigned char*)msg, iLength);

    return HR_OK;
}
```

## 延迟优化技术

### 1. CPU 缓存优化

**缓存行对齐**:

```cpp
// 避免伪共享（False Sharing）
struct alignas(64) MarketData {  // 64 字节 = CPU 缓存行大小
    char Ticker[32];
    double LastPrice;
    int Volume;
    // ... 其他字段
};

// 关键变量缓存行对齐
alignas(64) std::atomic<size_t> m_Head;
alignas(64) std::atomic<size_t> m_Tail;
```

**预取优化**:

```cpp
// 手动预取下一个数据
void ProcessMarketData(MarketData* data, int count) {
    for (int i = 0; i < count; i++) {
        // 预取下一个
        if (i + 1 < count) {
            __builtin_prefetch(&data[i + 1], 0, 3);
        }

        // 处理当前数据
        HandleMarketData(data[i]);
    }
}
```

### 2. 编译器优化

**CMake 配置**:

```cmake
# O3 优化级别
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")

# 针对本机 CPU 优化
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native -mtune=native")

# 启用链接时优化（LTO）
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")

# 内联优化
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -finline-functions")
```

**强制内联**:

```cpp
// 关键函数强制内联
__attribute__((always_inline))
inline void ConvertMarketData(const CTPData* in, MarketData* out) {
    // 数据转换逻辑
}
```

### 3. 系统调用优化

**减少系统调用**:

```cpp
// 批量处理，减少系统调用次数
void ProcessBatch() {
    Message::PackMessage batch[BATCH_SIZE];
    int count = 0;

    while (m_MarketMessageQueue.Pop(batch[count])) {
        count++;
        if (count >= BATCH_SIZE) break;
    }

    // 批量发送
    if (count > 0) {
        m_PubServer->PublishBatch(batch, count);
    }
}
```

**vDSO 时间函数**:

```cpp
// 使用 vDSO 加速时间获取（无系统调用）
static inline unsigned long getTimeUs() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);  // vDSO 优化
    return ts.tv_sec * 1000000UL + ts.tv_nsec / 1000;
}
```

## 总结

XMarketCenter 的技术实现体现了以下关键技术：

1. **插件架构**: 动态加载、接口统一、依赖注入
2. **无锁队列**: SPSC 模型、内存序、环形缓冲
3. **共享内存**: 零拷贝、发布-订阅、SPSC 通道
4. **网络通信**: Pack 协议、消息分发、异步发送
5. **性能优化**: 缓存对齐、预取、编译优化、系统优化

通过这些技术的综合运用，实现了微秒级的行情分发延迟。

**下一步**: 阅读 [插件开发指南](03_PluginDevelopment.md) 学习如何开发自定义插件。
