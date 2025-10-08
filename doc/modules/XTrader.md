# XTrader 模块文档

## 模块概述

XTrader 是 QuantFabric 交易系统的**交易网关**，负责接收报单/撤单请求，与风控系统交互，调用柜台 API 执行交易，并管理订单回报、仓位、资金信息。

**项目地址**: [XTrader](https://github.com/QuantFabric/XTrader)

## 核心功能

1. **接收交易请求**: 从 XQuant/HFTrader (共享内存) 和 XMonitor (网络) 接收报单/撤单请求
2. **风控交互**: 与 XRiskJudge 进行风控检查
3. **执行交易**: 调用 CTP/REM/YD/OES 等柜台 API 执行交易
4. **状态管理**: 管理订单状态、仓位、资金信息
5. **数据回传**: 将订单回报、仓位、资金写入共享内存，并转发至监控系统

## 架构设计

### 数据流向

```
┌─────────────┐         ┌──────────────┐
│   XQuant    │         │   XMonitor   │
└──────┬──────┘         └──────┬───────┘
       │ 共享内存              │ 网络
       ↓                        ↓
┌─────────────────────────────────────┐
│          TraderEngine               │
│  ┌──────────┐      ┌──────────────┐│
│  │WorkThread│      │MonitorThread ││
│  └────┬─────┘      └──────┬───────┘│
└───────┼────────────────────┼────────┘
        ↓                    ↓
   ┌─────────┐         ┌──────────┐
   │RiskJudge│         │ XWatcher │
   └────┬────┘         └──────────┘
        ↓
   ┌─────────────┐
   │TradeGateWay │ (插件)
   └──────┬──────┘
          ↓
    柜台 API (CTP/REM/YD/OES)
```

### 插件架构

XTrader 使用**插件模式**动态加载不同柜台交易网关：

```cpp
// 加载插件
TradeGateWay* m_TradeGateWay = XPluginEngine<TradeGateWay>::LoadPlugin(soPath, errorString);

// 执行报单
m_TradeGateWay->ReqInsertOrder(orderRequest);
```

### 线程模型

XTrader 采用**双线程模型**：

1. **WorkThread (工作线程)**:
   - 从 OrderServer 共享内存读取报单/撤单请求
   - 与 RiskServer 交互进行风控检查
   - 调用 TradeGateWay 执行交易
   - 处理订单回报

2. **MonitorThread (监控线程)**:
   - 从网络接收手动报单/撤单请求
   - 定时更新 App 状态
   - 转发订单回报到 XWatcher

## 核心组件

### 1. TraderEngine (交易引擎)

**文件**: `TraderEngine.cpp`、`TraderEngine.h`

**主要职责**:
- 加载配置和插件
- 管理线程
- 协调各组件工作

**启动流程**:
```cpp
1. LoadConfig()              // 加载配置
2. LoadTradeGateWay()        // 加载插件
3. Run()
   - RegisterClient()        // 连接 XWatcher
   - 连接 RiskServer         // 共享内存连接风控
   - 启动 OrderServer        // 共享内存服务
   - CreateTraderAPI()       // 创建柜台 API
   - LoadTrader()            // 登录柜台
   - Qry()                   // 查询仓位/资金
   - 启动 WorkThread         // 启动工作线程
   - 启动 MonitorThread      // 启动监控线程
```

**关键成员**:
```cpp
TradeGateWay* m_TradeGateWay;            // 交易网关插件
HPPackClient* m_HPPackClient;            // 连接 XWatcher
SHMConnection* m_RiskClient;             // 连接 RiskJudge
OrderServer* m_pOrderServer;             // OrderServer 共享内存服务
LockFreeQueue m_RequestMessageQueue;     // 请求队列
LockFreeQueue m_ReportMessageQueue;      // 回报队列
```

### 2. TradeGateWay (交易网关基类)

**文件**: `TraderAPI/TradeGateWay.hpp`

**抽象接口**:
```cpp
class TradeGateWay {
public:
    virtual void LoadAPIConfig() = 0;                   // 加载配置
    virtual void CreateTraderAPI() = 0;                 // 创建 API
    virtual void ReqUserLogin() = 0;                    // 登录
    virtual void ReqInsertOrder(...) = 0;               // 报单
    virtual void ReqCancelOrder(...) = 0;               // 撤单
    virtual int ReqQryFund() = 0;                       // 查询资金
    virtual int ReqQryPoistion() = 0;                   // 查询仓位
    virtual void UpdatePosition(...) = 0;               // 更新仓位
    virtual void UpdateFund(...) = 0;                   // 更新资金
};
```

**期货网关**: `FutureTradeGateWay.hpp`
- 继承 `TradeGateWay`
- 实现期货特定逻辑（开平仓、投机套保）

**股票网关**: `StockTradeGateWay.hpp`
- 继承 `TradeGateWay`
- 实现股票特定逻辑（买卖方向、两融）

### 3. OrderServer (共享内存服务)

**功能**:
- 接收 XQuant 的报单/撤单请求
- 提供 SPSC 队列

**实现**:
```cpp
class OrderServer : public SHMIPC::SHMServer<Message::PackMessage, ServerConf> {
    void Start(const std::string& name, int cpuset);
    void OnMessage(Message::PackMessage& msg);
};
```

### 4. HPPackClient (网络客户端)

**功能**:
- 连接到 XWatcher
- 接收手动报单/撤单请求
- 发送订单回报、仓位、资金到监控系统

## 插件实现

### CTP 插件

**文件**: `TraderAPI/CTP/CTPTradeGateWay.cpp`

**实现**:
```cpp
class CTPTradeGateWay : public FutureTradeGateWay, public CThostFtdcTraderSpi {
    void CreateTraderAPI() override {
        m_pTraderAPI = CThostFtdcTraderApi::CreateFtdcTraderApi();
        m_pTraderAPI->RegisterSpi(this);
        m_pTraderAPI->RegisterFront(m_FrontAddr);
        m_pTraderAPI->Init();
    }

    void ReqInsertOrder(const Message::TOrderRequest& request) override {
        CThostFtdcInputOrderField order;
        // 填充订单字段
        m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
    }

    // CTP 回调
    void OnRtnOrder(CThostFtdcOrderField *pOrder) override;
    void OnRtnTrade(CThostFtdcTradeField *pTrade) override;
};
```

**配置** (`CTP.yml`):
```yaml
CTPTrader:
  FrontTradeAddr: tcp://180.168.146.187:10130
  BrokerID: 9999
  Account: 123456
  Password: ******
  AppID: simnow_client_test
  AuthCode: 0000000000000000
```

### REM 插件

**文件**: `TraderAPI/REM/REMTradeGateWay.cpp`

**特点**:
- 低延迟交易
- 支持快速撤单

### YD 插件

**文件**: `TraderAPI/YD/YDTradeGateWay.cpp`

**特点**:
- **极低延迟** (纳秒级)
- 支持内核旁路
- HFTrader 商业版使用

### OES 插件

**文件**: `TraderAPI/OES/OESTradeGateWay.cpp`

**特点**:
- 股票交易
- 支持上交所、深交所

## 数据结构

### 报单请求 (TOrderRequest)

```cpp
struct TOrderRequest {
    char Account[16];          // 账户
    char Ticker[32];           // 合约代码
    char ExchangeID[16];       // 交易所
    int OrderToken;            // 订单 Token
    double OrderPrice;         // 报单价格
    int OrderVolume;           // 报单数量
    char OrderType;            // 订单类型 (限价/市价)
    char Direction;            // 买卖方向
    char Offset;               // 开平仓
    char HedgeFlag;            // 投机套保
    int RiskStatus;            // 风控状态
    unsigned long SendTimeStamp;  // 发送时间戳
};
```

### 撤单请求 (TActionRequest)

```cpp
struct TActionRequest {
    char Account[16];          // 账户
    char Ticker[32];           // 合约代码
    char ExchangeID[16];       // 交易所
    char OrderRef[32];         // 订单引用
    int OrderToken;            // 订单 Token
    int RiskStatus;            // 风控状态
};
```

### 订单状态 (TOrderStatus)

```cpp
struct TOrderStatus {
    char Account[16];          // 账户
    char Ticker[32];           // 合约代码
    char OrderRef[64];         // 订单引用
    int OrderToken;            // 订单 Token
    int OrderStatus;           // 订单状态
    double SendPrice;          // 报单价格
    int SendVolume;            // 报单数量
    int TotalTradedVolume;     // 成交数量
    double TradedAvgPrice;     // 成交均价
    int OrderSubmitStatus;     // 提交状态
    char OrderSysID[32];       // 柜台订单号
    unsigned long RecvMarketTime;   // 接收行情时间
    unsigned long SendOrderTime;    // 发送报单时间
    unsigned long InsertTime;       // 报单时间
    unsigned long CancelTime;       // 撤单时间
    unsigned long DoneTime;         // 完成时间
};
```

## 配置文件

### 主配置 (XTrader.yml)

```yaml
XTraderConfig:
  Product: Futures               # 产品类型 (Futures/Stock)
  BrokerID: 9999                 # 期货公司代码
  Account: 123456                # 账户
  AppID: simnow_client_test      # AppID
  AuthCode: 0000000000000000     # AuthCode
  ServerIP: 127.0.0.1            # XWatcher IP
  Port: 8001                     # XWatcher 端口
  OpenTime: 08:30:00             # 开盘时间
  CloseTime: 15:30:00            # 收盘时间
  TickerListPath: ./Config/TickerList.yml
  ErrorPath: ./Config/CTPError.yml
  RiskServerName: RiskServer     # 风控共享内存名
  OrderServerName: OrderServer   # 订单共享内存名
  TraderAPI: ./Config/CTP.yml    # 柜台 API 配置
  CPUSET: [10, 11]               # CPU 绑定 [WorkThread, OrderServer]
```

## 交易流程

### 报单流程（带风控）

```
1. XQuant 发送报单
   └─> OrderServer 共享内存

2. XTrader WorkThread 读取
   └─> 判断是否需要风控

3. 需要风控
   ├─> 发送到 RiskServer
   ├─> RiskJudge 检查
   └─> 返回检查结果

4. 风控通过
   ├─> TradeGateWay::ReqInsertOrder()
   └─> 调用柜台 API

5. 柜台回调
   ├─> OnRtnOrder() / OnRtnTrade()
   ├─> 更新订单状态
   ├─> 更新仓位/资金
   ├─> 写入共享内存 → XQuant
   └─> 发送网络 → XWatcher → XMonitor
```

### 报单流程（无风控）

```
1. XMonitor 发送手动报单
   └─> 网络 (HPPackClient)

2. XTrader MonitorThread 接收
   └─> 直接调用 TradeGateWay

3. 执行报单
   └─> 调用柜台 API

4. 回报处理
   └─> 同上
```

### 撤单流程

```
1. 接收撤单请求
   ├─> XQuant (共享内存)
   └─> XMonitor (网络)

2. 风控检查 (可选)
   └─> 检查撤单次数限制

3. 执行撤单
   └─> TradeGateWay::ReqCancelOrder()

4. 撤单回报
   └─> 更新订单状态
```

## 编译构建

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
PROJECT(XTrader)

set(CMAKE_CXX_STANDARD 17)
add_definitions(-D_GLIBCXX_USE_CXX11_ABI=0)

# 依赖库
include_directories(../Utils/)
include_directories(../SHMServer/)
include_directories(../XAPI/HP-Socket/5.8.2/include/)
include_directories(../XAPI/YAML-CPP/0.8.0/include/)
include_directories(../XAPI/FMTLogger/include/)
include_directories(../XAPI/parallel_hashmap/)

# 主程序
add_executable(XTrader_0.9.3
    main.cpp
    TraderEngine.cpp
    HPPackClient.cpp
)

target_link_libraries(XTrader_0.9.3
    pthread hpsocket4c yaml-cpp dl rt
)

# 插件编译
add_subdirectory(TraderAPI)
```

### 编译

```bash
cd QuantFabric
sh build_release.sh
# 输出: build/XTrader_0.9.3
#       build/libCTPTradeGateWay.so
```

## 运行方式

### 命令行启动

```bash
# 基本启动
./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so

# CPU 绑定
taskset -c 10 ./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so

# Onload 加速
onload ./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so
```

### 启动参数

```cpp
int main(int argc, char* argv[]) {
    // argv[1]: 配置文件路径
    // argv[2]: 插件 .so 文件路径
    TraderEngine engine;
    engine.LoadConfig(argv[1]);
    engine.LoadTradeGateWay(argv[2]);
    engine.Run();
}
```

## 性能优化

### 1. CPU 绑定
```cpp
Utils::ThreadBind(pthread_self(), cpuset);  // 绑定工作线程
m_pOrderServer->Start(name, cpuset);        // 绑定 OrderServer
```

### 2. 共享内存 IPC
- 零拷贝通信
- SPSC 无锁队列

### 3. 无锁队列
```cpp
LockFreeQueue<Message::PackMessage> m_RequestMessageQueue;
```

### 4. 插件动态加载
- 避免静态链接柜台 API
- 灵活切换柜台

## 监控与调试

### 日志

```bash
# 日志位置
./logs/XTrader_YYYYMMDD.log

# 关键日志
[INF] LoadTradeGateWay libCTPTradeGateWay.so successed
[INF] OnRspUserLogin successed, TradingDay=20250115
[INF] ReqInsertOrder Account:123456 Ticker:rb2505 Price:3500.0 Volume:1
[INF] OnRtnOrder OrderRef:1 OrderStatus:AllTraded
```

### 错误排查

**问题 1**: 连接柜台失败
```bash
# 检查网络
ping 180.168.146.187
telnet 180.168.146.187 10130

# 检查配置
cat Config/CTP.yml
```

**问题 2**: 报单被拒
```bash
# 查看错误码
cat Config/CTPError.yml | grep "错误码"

# 检查账户权限、资金、仓位
```

**问题 3**: 共享内存连接失败
```bash
# 检查 RiskJudge 是否运行
ps aux | grep RiskJudge

# 检查共享内存
ipcs -m | grep RiskServer
```

## 扩展开发

### 添加新柜台插件

1. **创建插件**:
```bash
mkdir TraderAPI/NewBroker
```

2. **实现接口**:
```cpp
class NewBrokerTradeGateWay : public TradeGateWay {
    void ReqInsertOrder(...) override {
        // 调用柜台 API
    }
    // 实现其他接口
};
```

3. **编译**:
```cmake
add_library(NewBrokerTradeGateWay SHARED
    NewBroker/NewBrokerTradeGateWay.cpp
)
target_link_libraries(NewBrokerTradeGateWay newbrokerapi)
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XTrader
