# XTrader 技术实现详解

## 插件加载机制

### 1. 动态库加载

XTrader 使用 `dlopen/dlsym` 动态加载交易网关插件。

**XPluginEngine 模板类**:
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

        // 2. 获取符号 "CreatePlugin"
        typedef T* (*CreatePluginFunc)();
        CreatePluginFunc createFunc = (CreatePluginFunc)dlsym(handle, "CreatePlugin");
        if (!createFunc) {
            errorString = "Symbol 'CreatePlugin' not found";
            dlclose(handle);
            return nullptr;
        }

        // 3. 创建插件实例
        T* plugin = createFunc();
        return plugin;
    }
};
```

**插件符号导出**:
```cpp
// CTPTradeGateWay.cpp
extern "C" {
    TradeGateWay* CreatePlugin() {
        return new CTPTradeGateWay();
    }
}
```

**使用示例**:
```cpp
// main.cpp
int main(int argc, char* argv[]) {
    std::string errorString;
    std::string soPath = argv[2];  // "libCTPTradeGateWay.so"

    // 加载插件
    TradeGateWay* gateway = XPluginEngine<TradeGateWay>::LoadPlugin(soPath, errorString);
    if (!gateway) {
        std::cerr << "Load plugin failed: " << errorString << std::endl;
        return -1;
    }

    // 使用插件
    gateway->SetLogger(logger);
    gateway->LoadAPIConfig();
    gateway->CreateTraderAPI();
    gateway->ReqUserLogin();
}
```

### 2. 插件配置加载

**YAML 配置**:
```yaml
# CTP.yml
CTPTrader:
  FrontTradeAddr: tcp://180.168.146.187:10130
  BrokerID: 9999
  Account: 123456
  Password: ******
  AppID: simnow_client_test
  AuthCode: 0000000000000000
```

**配置加载**:
```cpp
void CTPTradeGateWay::LoadAPIConfig() {
    Utils::YAMLConfig config;
    config.LoadConfig(m_ConfigPath);

    // 读取配置
    m_FrontAddr = config["CTPTrader"]["FrontTradeAddr"].as<std::string>();
    m_BrokerID = config["CTPTrader"]["BrokerID"].as<std::string>();
    m_Account = config["CTPTrader"]["Account"].as<std::string>();
    m_Password = config["CTPTrader"]["Password"].as<std::string>();
    m_AppID = config["CTPTrader"]["AppID"].as<std::string>();
    m_AuthCode = config["CTPTrader"]["AuthCode"].as<std::string>();
}
```

## 共享内存通信实现

### 1. OrderServer 架构

**OrderServer** 基于 SHMServer 实现，提供 SPSC 队列服务。

**定义**:
```cpp
class OrderServer : public SHMIPC::SHMServer<Message::PackMessage, ServerConf> {
public:
    void Start(const std::string& name, int cpuset) {
        // 设置共享内存名称
        SetServerName(name);

        // 绑定 CPU
        Utils::ThreadBind(pthread_self(), cpuset);

        // 创建共享内存
        CreateSharedMemory();

        // 启动接收线程
        StartReceiveThread();
    }

    void OnMessage(Message::PackMessage& msg) override {
        // 收到消息，推送到请求队列
        m_pTraderEngine->m_RequestMessageQueue.Push(msg);
    }

    void Broadcast(const Message::PackMessage& msg) {
        // 广播回报到所有连接的客户端
        for (auto& client : m_Clients) {
            client.second->PushBack(msg);
        }
    }

private:
    TraderEngine* m_pTraderEngine;
};
```

### 2. 内存布局

```
┌──────────────────────────────────────┐
│      OrderServer 共享内存布局         │
├──────────────────────────────────────┤
│  Control Block                       │
│  ┌────────────────────────────────┐ │
│  │ Magic Number                   │ │
│  │ Version                        │ │
│  │ Channel Count                  │ │
│  │ Max Clients                    │ │
│  └────────────────────────────────┘ │
├──────────────────────────────────────┤
│  Channel 0 (Server → Client 0)       │
│  ┌────────────────────────────────┐ │
│  │ SPSC Queue (PackMessage[1000]) │ │
│  │ head, tail (atomic)            │ │
│  └────────────────────────────────┘ │
├──────────────────────────────────────┤
│  Channel 1 (Client 0 → Server)       │
│  ┌────────────────────────────────┐ │
│  │ SPSC Queue (PackMessage[1000]) │ │
│  └────────────────────────────────┘ │
├──────────────────────────────────────┤
│  ... (更多客户端通道)                 │
└──────────────────────────────────────┘
```

### 3. SPSC 无锁队列实现

**队列结构**:
```cpp
template<typename T>
struct SPSCQueue {
    // 缓存行对齐，避免伪共享
    alignas(64) std::atomic<size_t> head;
    char padding1[64 - sizeof(std::atomic<size_t>)];

    alignas(64) std::atomic<size_t> tail;
    char padding2[64 - sizeof(std::atomic<size_t>)];

    size_t capacity;
    T buffer[0];  // 柔性数组
};
```

**Push 操作** (生产者):
```cpp
bool Push(const T& item) {
    size_t h = head.load(std::memory_order_relaxed);
    size_t next = (h + 1) % capacity;

    // 检查队列是否已满
    if (next == tail.load(std::memory_order_acquire)) {
        return false;  // 队列满
    }

    // 写入数据
    buffer[h] = item;

    // 更新 head (release 语义保证写入可见)
    head.store(next, std::memory_order_release);
    return true;
}
```

**Pop 操作** (消费者):
```cpp
bool Pop(T& item) {
    size_t t = tail.load(std::memory_order_relaxed);

    // 检查队列是否为空
    if (t == head.load(std::memory_order_acquire)) {
        return false;  // 队列空
    }

    // 读取数据
    item = buffer[t];

    // 更新 tail (release 语义)
    tail.store((t + 1) % capacity, std::memory_order_release);
    return true;
}
```

**内存序说明**:
- **relaxed**: 只保证原子性，无同步
- **acquire**: 读取时获取其他线程的写入
- **release**: 写入时释放，让其他线程可见

## 风控交互机制

### 1. RiskServer 连接

**SHMConnection 客户端**:
```cpp
// 连接 RiskServer
m_RiskClient = new SHMConnection("RiskServer");

// 发送风控请求
void SendRiskRequest(Message::PackMessage& msg) {
    msg.MessageType = Message::ERiskReport;
    msg.RiskReport.RiskStatus = Message::ECHECK_INIT;
    m_RiskClient->SendAsync(msg);
}

// 接收风控结果
void ReceiveRiskResult() {
    Message::PackMessage msg;
    if (m_RiskClient->PopFront(msg)) {
        if (msg.RiskReport.RiskStatus == Message::ECHECK_INIT) {
            // 风控通过
            m_TradeGateWay->ReqInsertOrder(msg.OrderRequest);
        } else {
            // 风控拒绝
            RejectOrder(msg);
        }
    }
}
```

### 2. 风控检查流程

```cpp
void ProcessOrder(Message::PackMessage& msg) {
    // 1. 判断是否需要风控
    if (NeedRiskCheck(msg)) {
        // 2. 设置风控状态
        msg.RiskReport.RiskStatus = Message::ECHECK_INIT;
        msg.RiskReport.CheckTimeStamp = Utils::getCurrentTimeUs();

        // 3. 发送到 RiskServer
        m_RiskClient->SendAsync(msg);

        // 4. 等待风控结果 (在 WorkThread 循环中)
    } else {
        // 无需风控，直接报单
        m_TradeGateWay->ReqInsertOrder(msg.OrderRequest);
    }
}

bool NeedRiskCheck(const Message::PackMessage& msg) {
    // 策略报单需要风控
    if (msg.MessageType == Message::EOrderRequest) {
        return true;
    }

    // 手动报单不需要风控
    if (msg.MessageType == Message::EManualOrder) {
        return false;
    }

    return false;
}
```

### 3. 风控数据结构

```cpp
struct TRiskReport {
    char Account[16];              // 账户
    char Ticker[32];               // 合约
    int RiskID;                    // 风控 ID
    int RiskStatus;                // 风控状态
    char RiskReason[256];          // 拒绝原因
    unsigned long CheckTimeStamp;  // 检查时间戳
    unsigned long DoneTimeStamp;   // 完成时间戳
};

enum RiskStatus {
    ECHECK_INIT     = 0,  // 初始化 (通过)
    ECHECK_CANCEL   = 1,  // 撤单限制
    ECHECK_FLOW     = 2,  // 流速限制
    ECHECK_SELF     = 3,  // 自成交
    ECHECK_LOCK     = 4,  // 账户锁定
    ECHECK_OTHER    = 99, // 其他
};
```

## 网络通信实现

### 1. HPPackClient 架构

**HPSocket Pack 协议**:
```
┌────────────────────────────────┐
│  消息格式                       │
├────────────────────────────────┤
│  4 bytes: 消息长度 (body size) │
│  N bytes: 消息体 (PackMessage) │
└────────────────────────────────┘
```

**客户端实现**:
```cpp
class HPPackClient : public IPackClientListener {
public:
    bool Connect(const std::string& ip, int port) {
        // 创建 Pack 客户端
        m_pPackClient = HP_Create_TcpPackClient(this);

        // 设置 Pack 长度头
        HP_TcpPackClient_SetMaxPackSize(m_pPackClient, 0xFFFFF);  // 1MB
        HP_TcpPackClient_SetPackHeaderFlag(m_pPackClient, 0x169);

        // 连接服务器
        return HP_Client_Start(m_pPackClient, ip.c_str(), port, FALSE);
    }

    virtual EnHandleResult OnConnect(IClient* pClient, CONNID dwConnID) override {
        m_Logger->info("Connected to XWatcher");

        // 注册客户端
        Message::PackMessage msg;
        msg.MessageType = Message::EAppRegister;
        strcpy(msg.AppRegister.Account, m_Account.c_str());
        msg.AppRegister.AppType = Message::EAppType::ETRADER;
        SendData(&msg, sizeof(msg));

        return HR_OK;
    }

    virtual EnHandleResult OnReceive(IClient* pClient, CONNID dwConnID,
                                     const BYTE* pData, int iLength) override {
        // 接收手动报单/撤单
        Message::PackMessage msg;
        memcpy(&msg, pData, sizeof(msg));

        // 推送到请求队列
        m_pTraderEngine->m_RequestMessageQueue.Push(msg);

        return HR_OK;
    }

    bool SendData(const void* pData, int iLength) {
        return HP_Client_Send(m_pPackClient, (BYTE*)pData, iLength);
    }

private:
    ITcpPackClient* m_pPackClient;
    TraderEngine* m_pTraderEngine;
};
```

### 2. 心跳机制

```cpp
void MonitorThread() {
    unsigned long lastHeartBeatTime = 0;

    while (m_Running) {
        unsigned long now = Utils::getCurrentTimeUs();

        // 每 30 秒发送心跳
        if (now - lastHeartBeatTime > 30 * 1000000) {
            Message::PackMessage msg;
            msg.MessageType = Message::EHeartBeat;
            msg.HeartBeat.Timestamp = now;
            m_HPPackClient->SendData(&msg, sizeof(msg));

            lastHeartBeatTime = now;
        }

        sleep(1);
    }
}
```

### 3. 重连机制

```cpp
void CheckConnection() {
    if (!m_HPPackClient->IsConnected()) {
        m_Logger->warn("Connection lost, reconnecting...");

        // 重连
        m_HPPackClient->Stop();
        sleep(5);
        m_HPPackClient->Connect(m_ServerIP, m_ServerPort);
    }
}
```

## 订单管理实现

### 1. OrderManager 类

```cpp
class OrderManager {
public:
    // 订单映射
    std::unordered_map<int, Message::TOrderStatus> m_OrderMap;  // OrderToken -> Order
    std::unordered_map<std::string, int> m_OrderRefMap;          // OrderRef -> OrderToken

    // 插入订单
    void InsertOrder(const Message::TOrderRequest& request) {
        Message::TOrderStatus order;
        order.OrderToken = request.OrderToken;
        strcpy(order.Account, request.Account);
        strcpy(order.Ticker, request.Ticker);
        order.SendPrice = request.OrderPrice;
        order.SendVolume = request.OrderVolume;
        order.OrderStatus = EORDER_PENDING;
        order.SendOrderTime = Utils::getCurrentTimeUs();

        m_OrderMap[order.OrderToken] = order;
    }

    // 更新订单
    void UpdateOrder(CThostFtdcOrderField* pOrder) {
        // 查找订单
        std::string orderRef = pOrder->OrderRef;
        auto it = m_OrderRefMap.find(orderRef);
        if (it == m_OrderRefMap.end()) {
            return;  // 订单不存在
        }

        int orderToken = it->second;
        auto& order = m_OrderMap[orderToken];

        // 更新订单状态
        order.OrderStatus = ConvertOrderStatus(pOrder->OrderStatus);
        strcpy(order.OrderSysID, pOrder->OrderSysID);
        order.InsertTime = ConvertTime(pOrder->InsertTime);

        // 推送更新
        BroadcastUpdate(order);
    }

    // 更新成交
    void UpdateTrade(CThostFtdcTradeField* pTrade) {
        std::string orderRef = pTrade->OrderRef;
        auto it = m_OrderRefMap.find(orderRef);
        if (it == m_OrderRefMap.end()) {
            return;
        }

        int orderToken = it->second;
        auto& order = m_OrderMap[orderToken];

        // 累计成交量
        order.TotalTradedVolume += pTrade->Volume;

        // 计算成交均价
        order.TradedAvgPrice = (order.TradedAvgPrice * (order.TotalTradedVolume - pTrade->Volume)
                               + pTrade->Price * pTrade->Volume) / order.TotalTradedVolume;

        // 更新状态
        if (order.TotalTradedVolume >= order.SendVolume) {
            order.OrderStatus = EORDER_ALL_TRADED;
            order.DoneTime = Utils::getCurrentTimeUs();
        } else {
            order.OrderStatus = EORDER_PARTIAL;
        }

        // 推送更新
        BroadcastUpdate(order);
    }

private:
    int ConvertOrderStatus(char status) {
        switch (status) {
            case THOST_FTDC_OST_AllTraded: return EORDER_ALL_TRADED;
            case THOST_FTDC_OST_PartTradedQueueing: return EORDER_PARTIAL;
            case THOST_FTDC_OST_NoTradeQueueing: return EORDER_ACCEPTED;
            case THOST_FTDC_OST_Canceled: return EORDER_CANCELED;
            default: return EORDER_ERROR;
        }
    }

    void BroadcastUpdate(const Message::TOrderStatus& order) {
        Message::PackMessage msg;
        msg.MessageType = Message::EOrderStatus;
        msg.OrderStatus = order;

        // 推送到共享内存
        m_pOrderServer->Broadcast(msg);

        // 推送到网络
        m_HPPackClient->SendData(&msg, sizeof(msg));
    }
};
```

### 2. 仓位管理

```cpp
class PositionManager {
public:
    // 仓位映射: Ticker -> Position
    std::unordered_map<std::string, Message::TAccountPosition> m_PositionMap;

    // 更新仓位 (查询)
    void UpdatePositionFromQuery(CThostFtdcInvestorPositionField* pPosition) {
        std::string key = pPosition->InstrumentID;
        auto& pos = m_PositionMap[key];

        strcpy(pos.Ticker, pPosition->InstrumentID);
        pos.LongPosition = pPosition->Position;  // 多头持仓
        pos.LongYdPosition = pPosition->YdPosition;  // 昨日持仓

        BroadcastPosition(pos);
    }

    // 更新仓位 (成交)
    void UpdatePositionFromTrade(CThostFtdcTradeField* pTrade) {
        std::string key = pTrade->InstrumentID;
        auto& pos = m_PositionMap[key];

        if (pTrade->Direction == THOST_FTDC_D_Buy) {
            // 买入
            if (pTrade->OffsetFlag == THOST_FTDC_OF_Open) {
                pos.LongPosition += pTrade->Volume;  // 开多
            } else {
                pos.ShortPosition -= pTrade->Volume;  // 平空
            }
        } else {
            // 卖出
            if (pTrade->OffsetFlag == THOST_FTDC_OF_Open) {
                pos.ShortPosition += pTrade->Volume;  // 开空
            } else {
                pos.LongPosition -= pTrade->Volume;  // 平多
            }
        }

        BroadcastPosition(pos);
    }

private:
    void BroadcastPosition(const Message::TAccountPosition& pos) {
        Message::PackMessage msg;
        msg.MessageType = Message::EAccountPosition;
        msg.AccountPosition = pos;

        m_pOrderServer->Broadcast(msg);
        m_HPPackClient->SendData(&msg, sizeof(msg));
    }
};
```

### 3. 资金管理

```cpp
class FundManager {
public:
    Message::TAccountFund m_Fund;

    // 更新资金 (查询)
    void UpdateFundFromQuery(CThostFtdcTradingAccountField* pAccount) {
        strcpy(m_Fund.Account, pAccount->AccountID);
        m_Fund.Balance = pAccount->Balance;                // 结算准备金
        m_Fund.Available = pAccount->Available;            // 可用资金
        m_Fund.Margin = pAccount->CurrMargin;              // 占用保证金
        m_Fund.FrozenMargin = pAccount->FrozenMargin;      // 冻结保证金
        m_Fund.Commission = pAccount->Commission;          // 手续费
        m_Fund.CloseProfit = pAccount->CloseProfit;        // 平仓盈亏
        m_Fund.PositionProfit = pAccount->PositionProfit;  // 持仓盈亏

        BroadcastFund();
    }

    // 更新资金 (成交)
    void UpdateFundFromTrade(CThostFtdcTradeField* pTrade) {
        // 扣除手续费 (简化计算)
        double commission = pTrade->Price * pTrade->Volume * 0.0001;
        m_Fund.Commission += commission;
        m_Fund.Available -= commission;

        BroadcastFund();
    }

private:
    void BroadcastFund() {
        Message::PackMessage msg;
        msg.MessageType = Message::EAccountFund;
        msg.AccountFund = m_Fund;

        m_pOrderServer->Broadcast(msg);
        m_HPPackClient->SendData(&msg, sizeof(msg));
    }
};
```

## CTP 插件实现

### 1. CTPTradeGateWay 类

```cpp
class CTPTradeGateWay : public FutureTradeGateWay, public CThostFtdcTraderSpi {
public:
    // ========== TradeGateWay 接口实现 ==========

    void CreateTraderAPI() override {
        m_pTraderAPI = CThostFtdcTraderApi::CreateFtdcTraderApi();
        m_pTraderAPI->RegisterSpi(this);
        m_pTraderAPI->RegisterFront(const_cast<char*>(m_FrontAddr.c_str()));
        m_pTraderAPI->SubscribePublicTopic(THOST_TERT_QUICK);
        m_pTraderAPI->SubscribePrivateTopic(THOST_TERT_QUICK);
        m_pTraderAPI->Init();
    }

    void ReqUserLogin() override {
        CThostFtdcReqUserLoginField req;
        memset(&req, 0, sizeof(req));
        strcpy(req.BrokerID, m_BrokerID.c_str());
        strcpy(req.UserID, m_Account.c_str());
        strcpy(req.Password, m_Password.c_str());
        strcpy(req.UserProductInfo, m_AppID.c_str());

        m_pTraderAPI->ReqUserLogin(&req, ++m_RequestID);
    }

    void ReqInsertOrder(const Message::TOrderRequest& request) override {
        CThostFtdcInputOrderField order;
        memset(&order, 0, sizeof(order));

        // 填充订单字段
        strcpy(order.BrokerID, m_BrokerID.c_str());
        strcpy(order.InvestorID, request.Account);
        strcpy(order.InstrumentID, request.Ticker);
        strcpy(order.ExchangeID, request.ExchangeID);

        // 订单引用
        sprintf(order.OrderRef, "%d", request.OrderToken);

        // 价格与数量
        order.LimitPrice = request.OrderPrice;
        order.VolumeTotalOriginal = request.OrderVolume;

        // 买卖方向
        order.Direction = ConvertDirection(request.Direction);

        // 开平仓
        order.CombOffsetFlag[0] = ConvertOffset(request.Offset);

        // 投机套保
        order.CombHedgeFlag[0] = ConvertHedgeFlag(request.HedgeFlag);

        // 订单类型
        if (request.OrderType == 'L') {
            order.OrderPriceType = THOST_FTDC_OPT_LimitPrice;  // 限价
        } else {
            order.OrderPriceType = THOST_FTDC_OPT_AnyPrice;    // 市价
        }

        // 时间条件
        order.TimeCondition = THOST_FTDC_TC_GFD;  // 当日有效

        // 成交量条件
        order.VolumeCondition = THOST_FTDC_VC_AV;  // 任意数量

        // 发送报单
        int ret = m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
        if (ret != 0) {
            m_Logger->error("ReqOrderInsert failed: {}", ret);
        }
    }

    void ReqCancelOrder(const Message::TActionRequest& request) override {
        CThostFtdcInputOrderActionField action;
        memset(&action, 0, sizeof(action));

        strcpy(action.BrokerID, m_BrokerID.c_str());
        strcpy(action.InvestorID, request.Account);
        strcpy(action.OrderRef, request.OrderRef);
        strcpy(action.InstrumentID, request.Ticker);
        strcpy(action.ExchangeID, request.ExchangeID);

        action.ActionFlag = THOST_FTDC_AF_Delete;  // 撤单

        int ret = m_pTraderAPI->ReqOrderAction(&action, ++m_RequestID);
        if (ret != 0) {
            m_Logger->error("ReqOrderAction failed: {}", ret);
        }
    }

    // ========== CTP 回调实现 ==========

    void OnRspUserLogin(CThostFtdcRspUserLoginField* pRspUserLogin,
                        CThostFtdcRspInfoField* pRspInfo, int nRequestID, bool bIsLast) override {
        if (pRspInfo && pRspInfo->ErrorID != 0) {
            m_Logger->error("Login failed: {} - {}", pRspInfo->ErrorID, pRspInfo->ErrorMsg);
            return;
        }

        m_Logger->info("Login success, TradingDay: {}", pRspUserLogin->TradingDay);
        m_TradingDay = pRspUserLogin->TradingDay;

        // 登录成功，查询结算确认
        ReqSettlementInfoConfirm();
    }

    void OnRtnOrder(CThostFtdcOrderField* pOrder) override {
        // 更新订单状态
        m_pOrderManager->UpdateOrder(pOrder);

        m_Logger->info("OnRtnOrder: OrderRef={} Status={} SysID={}",
                      pOrder->OrderRef, pOrder->OrderStatus, pOrder->OrderSysID);
    }

    void OnRtnTrade(CThostFtdcTradeField* pTrade) override {
        // 更新成交
        m_pOrderManager->UpdateTrade(pTrade);

        // 更新仓位
        m_pPositionManager->UpdatePositionFromTrade(pTrade);

        // 更新资金
        m_pFundManager->UpdateFundFromTrade(pTrade);

        m_Logger->info("OnRtnTrade: OrderRef={} Price={} Volume={}",
                      pTrade->OrderRef, pTrade->Price, pTrade->Volume);
    }

    void OnRspError(CThostFtdcRspInfoField* pRspInfo, int nRequestID, bool bIsLast) override {
        m_Logger->error("OnRspError: {} - {}", pRspInfo->ErrorID, pRspInfo->ErrorMsg);
    }

private:
    CThostFtdcTraderApi* m_pTraderAPI;
    OrderManager* m_pOrderManager;
    PositionManager* m_pPositionManager;
    FundManager* m_pFundManager;
};
```

### 2. 字段转换

```cpp
// 买卖方向转换
char ConvertDirection(char direction) {
    switch (direction) {
        case 'B': return THOST_FTDC_D_Buy;   // 买
        case 'S': return THOST_FTDC_D_Sell;  // 卖
        default: return THOST_FTDC_D_Buy;
    }
}

// 开平仓转换
char ConvertOffset(char offset) {
    switch (offset) {
        case 'O': return THOST_FTDC_OF_Open;        // 开仓
        case 'C': return THOST_FTDC_OF_Close;       // 平仓
        case 'T': return THOST_FTDC_OF_CloseToday;  // 平今
        case 'Y': return THOST_FTDC_OF_CloseYesterday;  // 平昨
        default: return THOST_FTDC_OF_Open;
    }
}

// 投机套保转换
char ConvertHedgeFlag(char flag) {
    switch (flag) {
        case 'S': return THOST_FTDC_HF_Speculation;  // 投机
        case 'A': return THOST_FTDC_HF_Arbitrage;    // 套利
        case 'H': return THOST_FTDC_HF_Hedge;        // 套保
        default: return THOST_FTDC_HF_Speculation;
    }
}
```

## 性能优化技术

### 1. CPU 缓存优化

```cpp
// 缓存行对齐
alignas(64) struct OrderData {
    int orderToken;
    double price;
    int volume;
    // ...
};

// 预取数据
void ProcessOrders() {
    for (size_t i = 0; i < orders.size(); ++i) {
        // 预取下一个订单到缓存
        if (i + 1 < orders.size()) {
            __builtin_prefetch(&orders[i + 1], 0, 3);
        }

        ProcessOrder(orders[i]);
    }
}
```

### 2. 编译器优化

```cpp
// 强制内联
inline __attribute__((always_inline))
void FastProcessOrder(const Message::TOrderRequest& order) {
    // 关键路径代码
}

// 分支预测
if (__builtin_expect(order.NeedRiskCheck, 1)) {
    // 常见分支
    CheckRisk(order);
} else {
    // 不常见分支
    DirectInsert(order);
}
```

### 3. 系统调用优化

```cpp
// 使用 clock_gettime (VDSO)
unsigned long getCurrentTimeUs() {
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    return ts.tv_sec * 1000000 + ts.tv_nsec / 1000;
}

// 批量网络发送
std::vector<Message::PackMessage> batch;
batch.reserve(100);

while (m_ReportMessageQueue.Pop(msg)) {
    batch.push_back(msg);
    if (batch.size() >= 100) break;
}

for (auto& msg : batch) {
    m_HPPackClient->SendData(&msg, sizeof(msg));
}
```

## 总结

XTrader 核心技术实现：

1. **插件加载**: dlopen/dlsym 动态加载，灵活切换柜台
2. **共享内存**: SPSC 无锁队列，1-3μs 通信延迟
3. **风控集成**: 与 RiskServer 实时交互，订单前置检查
4. **网络通信**: HPSocket Pack 协议，可靠传输
5. **订单管理**: 精确状态机，支持仓位/资金管理
6. **性能优化**: CPU 绑定、缓存优化、批量处理

**关键优势**:
- 极致低延迟 (< 20μs)
- 多柜台支持 (CTP/REM/YD/OES)
- 风控集成
- 高可靠性

---
作者: @scorpiostudio @yutiansut
