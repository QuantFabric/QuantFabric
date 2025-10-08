# XTrader 插件开发指南

## 插件开发流程

### 完整步骤

1. **创建插件目录**
2. **实现 TradeGateWay 接口**
3. **编写配置加载逻辑**
4. **实现柜台 API 回调**
5. **编译配置 (CMakeLists.txt)**
6. **测试与调试**

## 步骤 1: 创建插件目录

```bash
cd QuantFabric/XTrader/TraderAPI
mkdir NewBroker
cd NewBroker
```

**文件结构**:
```
NewBroker/
├── NewBrokerTradeGateWay.h          # 头文件
├── NewBrokerTradeGateWay.cpp        # 实现文件
├── NewBroker.yml                    # 配置文件
└── CMakeLists.txt                   # 编译配置
```

## 步骤 2: 实现 TradeGateWay 接口

### 2.1 头文件 (NewBrokerTradeGateWay.h)

```cpp
#ifndef NEWBROKER_TRADE_GATEWAY_H
#define NEWBROKER_TRADE_GATEWAY_H

#include "../TradeGateWay.hpp"
#include "NewBrokerAPI.h"  // 柜台 API 头文件

class NewBrokerTradeGateWay : public FutureTradeGateWay,
                               public NewBrokerTraderSpi {
public:
    NewBrokerTradeGateWay();
    virtual ~NewBrokerTradeGateWay();

    // ========== TradeGateWay 接口 ==========
    virtual void LoadAPIConfig() override;
    virtual void CreateTraderAPI() override;
    virtual void ReqUserLogin() override;
    virtual void ReqInsertOrder(const Message::TOrderRequest&) override;
    virtual void ReqCancelOrder(const Message::TActionRequest&) override;
    virtual int ReqQryFund() override;
    virtual int ReqQryPoistion() override;
    virtual void Release() override;

    // ========== 柜台 API 回调 ==========
    virtual void OnConnected() override;
    virtual void OnDisconnected() override;
    virtual void OnRspLogin(LoginRsp* pRsp, int nRequestID) override;
    virtual void OnRspOrderInsert(OrderRsp* pRsp, int nRequestID) override;
    virtual void OnRtnOrder(OrderField* pOrder) override;
    virtual void OnRtnTrade(TradeField* pTrade) override;

private:
    // 柜台 API 实例
    NewBrokerTraderAPI* m_pTraderAPI;

    // 配置参数
    std::string m_FrontAddr;
    std::string m_BrokerID;
    std::string m_Account;
    std::string m_Password;

    // 请求 ID
    int m_RequestID;

    // 辅助方法
    char ConvertDirection(char direction);
    char ConvertOffset(char offset);
    int ConvertOrderStatus(char status);
};

// 导出符号
extern "C" {
    TradeGateWay* CreatePlugin();
}

#endif
```

### 2.2 实现文件 (NewBrokerTradeGateWay.cpp)

```cpp
#include "NewBrokerTradeGateWay.h"
#include "../../Utils/Logger.h"
#include <cstring>

NewBrokerTradeGateWay::NewBrokerTradeGateWay()
    : m_pTraderAPI(nullptr), m_RequestID(0) {
}

NewBrokerTradeGateWay::~NewBrokerTradeGateWay() {
    Release();
}

// ========== 配置加载 ==========
void NewBrokerTradeGateWay::LoadAPIConfig() {
    Utils::YAMLConfig config;
    config.LoadConfig(m_ConfigPath);

    m_FrontAddr = config["NewBrokerTrader"]["FrontAddr"].as<std::string>();
    m_BrokerID = config["NewBrokerTrader"]["BrokerID"].as<std::string>();
    m_Account = config["NewBrokerTrader"]["Account"].as<std::string>();
    m_Password = config["NewBrokerTrader"]["Password"].as<std::string>();

    m_Logger->info("LoadAPIConfig: FrontAddr={} Account={}", m_FrontAddr, m_Account);
}

// ========== API 创建 ==========
void NewBrokerTradeGateWay::CreateTraderAPI() {
    // 创建柜台 API
    m_pTraderAPI = NewBrokerTraderAPI::CreateAPI();

    // 注册回调
    m_pTraderAPI->RegisterSpi(this);

    // 设置地址
    m_pTraderAPI->RegisterFront(const_cast<char*>(m_FrontAddr.c_str()));

    // 初始化
    m_pTraderAPI->Init();

    m_Logger->info("CreateTraderAPI success");
}

// ========== 登录 ==========
void NewBrokerTradeGateWay::ReqUserLogin() {
    NewBrokerLoginReq req;
    memset(&req, 0, sizeof(req));

    strcpy(req.BrokerID, m_BrokerID.c_str());
    strcpy(req.UserID, m_Account.c_str());
    strcpy(req.Password, m_Password.c_str());

    int ret = m_pTraderAPI->ReqUserLogin(&req, ++m_RequestID);
    if (ret != 0) {
        m_Logger->error("ReqUserLogin failed: {}", ret);
    }
}

// ========== 报单 ==========
void NewBrokerTradeGateWay::ReqInsertOrder(const Message::TOrderRequest& request) {
    NewBrokerInputOrder order;
    memset(&order, 0, sizeof(order));

    // 基本信息
    strcpy(order.BrokerID, m_BrokerID.c_str());
    strcpy(order.InvestorID, request.Account);
    strcpy(order.InstrumentID, request.Ticker);
    strcpy(order.ExchangeID, request.ExchangeID);

    // 订单引用
    sprintf(order.OrderRef, "%d", request.OrderToken);

    // 价格与数量
    order.LimitPrice = request.OrderPrice;
    order.Volume = request.OrderVolume;

    // 买卖方向
    order.Direction = ConvertDirection(request.Direction);

    // 开平仓
    order.OffsetFlag = ConvertOffset(request.Offset);

    // 订单类型
    if (request.OrderType == 'L') {
        order.OrderPriceType = OrderPriceType_Limit;  // 限价
    } else {
        order.OrderPriceType = OrderPriceType_Market; // 市价
    }

    // 发送报单
    int ret = m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
    if (ret != 0) {
        m_Logger->error("ReqOrderInsert failed: {}", ret);
    } else {
        m_Logger->info("ReqOrderInsert: Ticker={} Price={} Volume={}",
                      request.Ticker, request.OrderPrice, request.OrderVolume);
    }
}

// ========== 撤单 ==========
void NewBrokerTradeGateWay::ReqCancelOrder(const Message::TActionRequest& request) {
    NewBrokerOrderAction action;
    memset(&action, 0, sizeof(action));

    strcpy(action.BrokerID, m_BrokerID.c_str());
    strcpy(action.InvestorID, request.Account);
    strcpy(action.OrderRef, request.OrderRef);
    strcpy(action.InstrumentID, request.Ticker);
    strcpy(action.ExchangeID, request.ExchangeID);

    action.ActionFlag = ActionFlag_Delete;  // 撤单

    int ret = m_pTraderAPI->ReqOrderAction(&action, ++m_RequestID);
    if (ret != 0) {
        m_Logger->error("ReqOrderAction failed: {}", ret);
    }
}

// ========== 查询资金 ==========
int NewBrokerTradeGateWay::ReqQryFund() {
    NewBrokerQryAccount req;
    memset(&req, 0, sizeof(req));

    strcpy(req.BrokerID, m_BrokerID.c_str());
    strcpy(req.InvestorID, m_Account.c_str());

    return m_pTraderAPI->ReqQryTradingAccount(&req, ++m_RequestID);
}

// ========== 查询仓位 ==========
int NewBrokerTradeGateWay::ReqQryPoistion() {
    NewBrokerQryPosition req;
    memset(&req, 0, sizeof(req));

    strcpy(req.BrokerID, m_BrokerID.c_str());
    strcpy(req.InvestorID, m_Account.c_str());

    return m_pTraderAPI->ReqQryInvestorPosition(&req, ++m_RequestID);
}

// ========== 释放资源 ==========
void NewBrokerTradeGateWay::Release() {
    if (m_pTraderAPI) {
        m_pTraderAPI->Release();
        m_pTraderAPI = nullptr;
    }
}

// ========== 柜台回调 ==========

void NewBrokerTradeGateWay::OnConnected() {
    m_Logger->info("OnConnected");
}

void NewBrokerTradeGateWay::OnDisconnected() {
    m_Logger->warn("OnDisconnected");
}

void NewBrokerTradeGateWay::OnRspLogin(LoginRsp* pRsp, int nRequestID) {
    if (pRsp->ErrorID != 0) {
        m_Logger->error("Login failed: {} - {}", pRsp->ErrorID, pRsp->ErrorMsg);
        return;
    }

    m_Logger->info("Login success, TradingDay: {}", pRsp->TradingDay);
    m_TradingDay = pRsp->TradingDay;

    // 登录成功后，查询仓位和资金
    ReqQryPoistion();
    ReqQryFund();
}

void NewBrokerTradeGateWay::OnRspOrderInsert(OrderRsp* pRsp, int nRequestID) {
    if (pRsp->ErrorID != 0) {
        m_Logger->error("OrderInsert failed: {} - {}", pRsp->ErrorID, pRsp->ErrorMsg);

        // 构造拒绝回报
        Message::TOrderStatus order;
        order.OrderToken = atoi(pRsp->OrderRef);
        order.OrderStatus = Message::EORDER_REJECTED;
        strcpy(order.ErrorMsg, pRsp->ErrorMsg);

        UpdateOrderStatus(order);
    }
}

void NewBrokerTradeGateWay::OnRtnOrder(OrderField* pOrder) {
    // 更新订单状态
    Message::TOrderStatus order;
    order.OrderToken = atoi(pOrder->OrderRef);
    order.OrderStatus = ConvertOrderStatus(pOrder->OrderStatus);
    strcpy(order.Account, pOrder->InvestorID);
    strcpy(order.Ticker, pOrder->InstrumentID);
    strcpy(order.OrderSysID, pOrder->OrderSysID);
    order.SendPrice = pOrder->LimitPrice;
    order.SendVolume = pOrder->Volume;
    order.TotalTradedVolume = pOrder->VolumeTraded;

    UpdateOrderStatus(order);

    m_Logger->info("OnRtnOrder: OrderRef={} Status={} SysID={}",
                  pOrder->OrderRef, pOrder->OrderStatus, pOrder->OrderSysID);
}

void NewBrokerTradeGateWay::OnRtnTrade(TradeField* pTrade) {
    // 更新成交
    Message::TOrderStatus trade;
    trade.OrderToken = atoi(pTrade->OrderRef);
    trade.TradedVolume = pTrade->Volume;
    trade.TradedPrice = pTrade->Price;

    UpdateOrderStatus(trade);

    // 更新仓位
    UpdatePosition(...);

    m_Logger->info("OnRtnTrade: OrderRef={} Price={} Volume={}",
                  pTrade->OrderRef, pTrade->Price, pTrade->Volume);
}

// ========== 辅助方法 ==========

char NewBrokerTradeGateWay::ConvertDirection(char direction) {
    switch (direction) {
        case 'B': return Direction_Buy;
        case 'S': return Direction_Sell;
        default: return Direction_Buy;
    }
}

char NewBrokerTradeGateWay::ConvertOffset(char offset) {
    switch (offset) {
        case 'O': return OffsetFlag_Open;
        case 'C': return OffsetFlag_Close;
        case 'T': return OffsetFlag_CloseToday;
        case 'Y': return OffsetFlag_CloseYesterday;
        default: return OffsetFlag_Open;
    }
}

int NewBrokerTradeGateWay::ConvertOrderStatus(char status) {
    switch (status) {
        case OrderStatus_AllTraded: return Message::EORDER_ALL_TRADED;
        case OrderStatus_PartTraded: return Message::EORDER_PARTIAL;
        case OrderStatus_NoTrade: return Message::EORDER_ACCEPTED;
        case OrderStatus_Canceled: return Message::EORDER_CANCELED;
        default: return Message::EORDER_ERROR;
    }
}

// ========== 导出符号 ==========
extern "C" {
    TradeGateWay* CreatePlugin() {
        return new NewBrokerTradeGateWay();
    }
}
```

## 步骤 3: 配置文件 (NewBroker.yml)

```yaml
NewBrokerTrader:
  FrontAddr: tcp://192.168.1.100:8888
  BrokerID: 1000
  Account: test_account
  Password: test_password
```

## 步骤 4: 编译配置 (CMakeLists.txt)

### 4.1 TraderAPI/CMakeLists.txt (添加子目录)

```cmake
# 添加 NewBroker 子目录
add_subdirectory(NewBroker)
```

### 4.2 TraderAPI/NewBroker/CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

# 柜台 API 路径
set(NewBroker_API_PATH ${CMAKE_SOURCE_DIR}/XAPI/NewBroker)

# 头文件路径
include_directories(${NewBroker_API_PATH}/include)
include_directories(${CMAKE_SOURCE_DIR}/Utils)
include_directories(${CMAKE_SOURCE_DIR}/XAPI/YAML-CPP/0.8.0/include)

# 链接库路径
link_directories(${NewBroker_API_PATH}/lib)

# 编译动态库
add_library(NewBrokerTradeGateWay SHARED
    NewBrokerTradeGateWay.cpp
)

# 链接库
target_link_libraries(NewBrokerTradeGateWay
    newbrokerapi    # 柜台 API 库
    yaml-cpp
    pthread
)

# 输出到 build 目录
set_target_properties(NewBrokerTradeGateWay PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}
)
```

## 步骤 5: 编译与测试

### 5.1 编译

```bash
cd QuantFabric
sh build_release.sh

# 检查输出
ls build/libNewBrokerTradeGateWay.so
```

### 5.2 配置 XTrader

**XTrader.yml**:
```yaml
XTraderConfig:
  Product: Futures
  BrokerID: 1000
  Account: test_account
  # ...
  TraderAPI: ./Config/NewBroker.yml  # 指向插件配置
```

### 5.3 运行测试

```bash
cd build

# 启动 XTrader 并加载插件
./XTrader_0.9.3 ../Config/XTrader.yml ./libNewBrokerTradeGateWay.so
```

### 5.4 查看日志

```bash
tail -f logs/XTrader_20250108.log
```

**预期日志**:
```
[INF] LoadTradeGateWay libNewBrokerTradeGateWay.so success
[INF] LoadAPIConfig: FrontAddr=tcp://192.168.1.100:8888 Account=test_account
[INF] CreateTraderAPI success
[INF] OnConnected
[INF] Login success, TradingDay: 20250108
[INF] ReqQryPosition success
[INF] ReqQryFund success
```

## 步骤 6: 调试技巧

### 6.1 使用 GDB 调试

```bash
gdb --args ./XTrader_0.9.3 ../Config/XTrader.yml ./libNewBrokerTradeGateWay.so

(gdb) break NewBrokerTradeGateWay::ReqInsertOrder
(gdb) run
(gdb) bt
(gdb) print request
```

### 6.2 日志调试

```cpp
// 添加详细日志
m_Logger->debug("ReqInsertOrder: Ticker={} Price={} Volume={} Direction={} Offset={}",
               request.Ticker, request.OrderPrice, request.OrderVolume,
               request.Direction, request.Offset);

// 打印柜台 API 返回值
int ret = m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
m_Logger->debug("ReqOrderInsert return: {}", ret);
```

### 6.3 内存检查

```bash
valgrind --leak-check=full ./XTrader_0.9.3 ../Config/XTrader.yml ./libNewBrokerTradeGateWay.so
```

## 最佳实践

### 1. 错误处理

```cpp
// 检查 API 返回值
int ret = m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
if (ret != 0) {
    m_Logger->error("ReqOrderInsert failed: {}", ret);

    // 返回拒绝回报
    Message::TOrderStatus order;
    order.OrderToken = request.OrderToken;
    order.OrderStatus = Message::EORDER_REJECTED;
    sprintf(order.ErrorMsg, "API error: %d", ret);
    UpdateOrderStatus(order);
}
```

### 2. 线程安全

```cpp
// 使用互斥锁保护共享数据
std::mutex m_Mutex;

void UpdateOrderMap(int orderToken, const Message::TOrderStatus& order) {
    std::lock_guard<std::mutex> lock(m_Mutex);
    m_OrderMap[orderToken] = order;
}
```

### 3. 性能优化

```cpp
// 避免频繁内存分配
class NewBrokerTradeGateWay {
private:
    // 预分配缓冲区
    char m_Buffer[1024];

    // 复用对象
    NewBrokerInputOrder m_OrderBuffer;
};

void ReqInsertOrder(const Message::TOrderRequest& request) {
    // 复用 m_OrderBuffer，避免每次分配
    memset(&m_OrderBuffer, 0, sizeof(m_OrderBuffer));
    // 填充字段...
    m_pTraderAPI->ReqOrderInsert(&m_OrderBuffer, ++m_RequestID);
}
```

### 4. 资源管理

```cpp
// RAII 管理 API 资源
class NewBrokerTradeGateWay {
public:
    ~NewBrokerTradeGateWay() {
        Release();
    }

    void Release() {
        if (m_pTraderAPI) {
            m_pTraderAPI->RegisterSpi(nullptr);  // 注销回调
            m_pTraderAPI->Release();              // 释放 API
            m_pTraderAPI = nullptr;
        }
    }
};
```

### 5. 配置验证

```cpp
void NewBrokerTradeGateWay::LoadAPIConfig() {
    Utils::YAMLConfig config;
    config.LoadConfig(m_ConfigPath);

    // 验证必需字段
    if (!config["NewBrokerTrader"]["FrontAddr"]) {
        throw std::runtime_error("FrontAddr not found in config");
    }

    if (!config["NewBrokerTrader"]["Account"]) {
        throw std::runtime_error("Account not found in config");
    }

    m_FrontAddr = config["NewBrokerTrader"]["FrontAddr"].as<std::string>();
    m_Account = config["NewBrokerTrader"]["Account"].as<std::string>();

    // 验证格式
    if (m_FrontAddr.find("tcp://") != 0) {
        throw std::runtime_error("Invalid FrontAddr format");
    }
}
```

## 常见问题排查

### 问题 1: 插件加载失败

**错误信息**:
```
LoadTradeGateWay failed: Symbol 'CreatePlugin' not found
```

**解决方法**:
```cpp
// 检查是否使用 extern "C" 导出符号
extern "C" {
    TradeGateWay* CreatePlugin() {
        return new NewBrokerTradeGateWay();
    }
}

// 检查编译输出
nm -D libNewBrokerTradeGateWay.so | grep CreatePlugin
```

### 问题 2: 柜台 API 连接失败

**错误信息**:
```
OnDisconnected
```

**排查步骤**:
```bash
# 1. 检查网络
ping 192.168.1.100
telnet 192.168.1.100 8888

# 2. 检查防火墙
sudo iptables -L

# 3. 检查柜台服务是否运行
```

### 问题 3: 报单被拒

**错误信息**:
```
OrderInsert failed: -3 - CTP:不合法的合约
```

**排查步骤**:
```bash
# 1. 检查合约代码是否正确
# 2. 检查交易所代码是否正确
# 3. 检查是否在交易时间
# 4. 查看柜台错误码文档
```

### 问题 4: 内存泄漏

**使用 Valgrind 检测**:
```bash
valgrind --leak-check=full --show-leak-kinds=all ./XTrader_0.9.3 ...
```

**常见原因**:
```cpp
// 未释放 API
void NewBrokerTradeGateWay::Release() {
    if (m_pTraderAPI) {
        m_pTraderAPI->Release();  // 必须调用
        m_pTraderAPI = nullptr;
    }
}
```

## 示例：REM 插件精简版

```cpp
// REMTradeGateWay.h
class REMTradeGateWay : public FutureTradeGateWay, public CRZQThostFtdcTraderSpi {
public:
    void CreateTraderAPI() override {
        m_pTraderAPI = CRZQThostFtdcTraderApi::CreateFtdcTraderApi();
        m_pTraderAPI->RegisterSpi(this);
        m_pTraderAPI->RegisterFront(const_cast<char*>(m_FrontAddr.c_str()));
        m_pTraderAPI->Init();
    }

    void ReqInsertOrder(const Message::TOrderRequest& request) override {
        CRZQThostFtdcInputOrderField order;
        memset(&order, 0, sizeof(order));
        // 填充字段...
        m_pTraderAPI->ReqOrderInsert(&order, ++m_RequestID);
    }

    void OnRtnOrder(CRZQThostFtdcOrderField* pOrder) override {
        // 处理订单回报
        UpdateOrderStatus(...);
    }

private:
    CRZQThostFtdcTraderApi* m_pTraderAPI;
};

extern "C" {
    TradeGateWay* CreatePlugin() {
        return new REMTradeGateWay();
    }
}
```

## 总结

插件开发关键步骤：

1. **实现接口**: 继承 TradeGateWay，实现所有虚函数
2. **符号导出**: 使用 `extern "C"` 导出 CreatePlugin
3. **柜台适配**: 转换数据结构，适配柜台 API
4. **错误处理**: 处理 API 错误，返回拒绝回报
5. **资源管理**: RAII 模式，确保资源释放
6. **测试验证**: 单元测试、集成测试、压力测试

**开发建议**:
- 参考 CTP 插件实现
- 详细日志，便于调试
- 错误处理要完善
- 性能优化要考虑

---
作者: @scorpiostudio @yutiansut
