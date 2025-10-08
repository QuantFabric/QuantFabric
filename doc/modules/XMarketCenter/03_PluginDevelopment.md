# XMarketCenter 插件开发指南

## 插件开发流程

### 步骤 1: 创建插件目录

```bash
cd XMarketCenter/MarketAPI
mkdir NewBroker
cd NewBroker
```

### 步骤 2: 实现插件类

**NewBrokerMarketGateWay.h**:

```cpp
#ifndef NEWBROKERMARKETGATEWAY_H
#define NEWBROKERMARKETGATEWAY_H

#include "MarketGateWay.hpp"
#include "NewBrokerMdApi.h"  // 柜台 API 头文件

class NewBrokerMarketGateWay : public MarketGateWay,
                                public NewBrokerMdSpi {  // 柜台回调接口
public:
    NewBrokerMarketGateWay();
    ~NewBrokerMarketGateWay();

    // ========== MarketGateWay 接口实现 ==========
    bool LoadAPIConfig() override;
    void Run() override;
    void GetCommitID(std::string& CommitID, std::string& UtilsCommitID) override;
    void GetAPIVersion(std::string& APIVersion) override;

    // ========== 柜台回调函数 ==========
    void OnConnect() override;
    void OnDisconnect(int reason) override;
    void OnLogin(int errorCode, const char* errorMsg) override;
    void OnMarketData(NewBrokerMarketData* pData) override;

private:
    void CreateAPI();
    void ReqUserLogin();
    void SubscribeMarketData();
    void ConvertMarketData(NewBrokerMarketData* pData,
                          MarketData::TFutureMarketData& out);

    NewBrokerMdApi* m_pMdApi;
    std::string m_ServerAddr;
    std::string m_Account;
    std::string m_Password;
    int m_ConnectedStatus;
    unsigned int m_TickCount;
};

// 插件导出函数
extern "C" {
    MarketGateWay* CreatePlugin();
}

#endif
```

**NewBrokerMarketGateWay.cpp**:

```cpp
#include "NewBrokerMarketGateWay.h"

NewBrokerMarketGateWay::NewBrokerMarketGateWay()
    : m_pMdApi(nullptr),
      m_ConnectedStatus(0),
      m_TickCount(0) {
}

NewBrokerMarketGateWay::~NewBrokerMarketGateWay() {
    if (m_pMdApi) {
        m_pMdApi->Release();
        m_pMdApi = nullptr;
    }
}

// ========== 1. 加载配置 ==========
bool NewBrokerMarketGateWay::LoadAPIConfig() {
    std::string errorString;

    // 加载柜台特定配置
    NewBrokerConfig config;
    if (!Utils::LoadNewBrokerConfig(
            m_MarketCenterConfig.APIConfig.c_str(),
            config, errorString)) {
        m_Logger->Log->error("LoadAPIConfig failed: {}", errorString);
        return false;
    }

    m_ServerAddr = config.ServerAddr;
    m_Account = config.Account;
    m_Password = config.Password;

    m_Logger->Log->info("LoadAPIConfig success: Server={}, Account={}",
                        m_ServerAddr, m_Account);
    return true;
}

// ========== 2. 启动插件 ==========
void NewBrokerMarketGateWay::Run() {
    m_Logger->Log->info("NewBrokerMarketGateWay starting...");

    // 创建 API
    CreateAPI();

    // 连接服务器（阻塞或异步，取决于柜台 API）
    m_pMdApi->Connect(m_ServerAddr.c_str());

    // Join 等待（如果 API 是多线程的）
    m_pMdApi->Join();
}

// ========== 3. 获取版本信息 ==========
void NewBrokerMarketGateWay::GetCommitID(std::string& CommitID,
                                          std::string& UtilsCommitID) {
    CommitID = APP_COMMITID;           // 插件 CommitID
    UtilsCommitID = UTILS_COMMITID;    // Utils CommitID
}

void NewBrokerMarketGateWay::GetAPIVersion(std::string& APIVersion) {
    APIVersion = m_pMdApi->GetVersion();  // 柜台 API 版本
}

// ========== 4. 创建 API ==========
void NewBrokerMarketGateWay::CreateAPI() {
    m_pMdApi = NewBrokerMdApi::CreateApi();
    m_pMdApi->RegisterSpi(this);  // 注册回调

    m_Logger->Log->info("API created, version={}", m_pMdApi->GetVersion());
}

// ========== 5. 柜台回调：连接成功 ==========
void NewBrokerMarketGateWay::OnConnect() {
    m_ConnectedStatus = 1;
    m_Logger->Log->info("OnConnect: Connected to {}", m_ServerAddr);

    // 发送登录请求
    ReqUserLogin();
}

// ========== 6. 柜台回调：断线 ==========
void NewBrokerMarketGateWay::OnDisconnect(int reason) {
    m_ConnectedStatus = 0;
    m_Logger->Log->warn("OnDisconnect: reason={}", reason);

    // 自动重连
    sleep(5);
    m_pMdApi->Connect(m_ServerAddr.c_str());
}

// ========== 7. 柜台回调：登录响应 ==========
void NewBrokerMarketGateWay::OnLogin(int errorCode, const char* errorMsg) {
    if (errorCode != 0) {
        m_Logger->Log->error("OnLogin failed: code={}, msg={}", errorCode, errorMsg);
        return;
    }

    m_Logger->Log->info("OnLogin success");

    // 订阅行情
    SubscribeMarketData();
}

// ========== 8. 柜台回调：行情数据 ==========
void NewBrokerMarketGateWay::OnMarketData(NewBrokerMarketData* pData) {
    // 过滤不需要的合约
    if (m_TickerExchangeMap.find(pData->InstrumentID) == m_TickerExchangeMap.end()) {
        return;
    }

    // 构造消息
    Message::PackMessage message;
    memset(&message, 0, sizeof(message));
    message.MessageType = Message::EMessageType::EFutureMarket;

    // 数据转换
    ConvertMarketData(pData, message.FutureMarketData);

    // 写入无锁队列（主程序会读取）
    m_MarketMessageQueue.Push(message);
}

// ========== 9. 登录请求 ==========
void NewBrokerMarketGateWay::ReqUserLogin() {
    NewBrokerLoginReq req;
    strncpy(req.Account, m_Account.c_str(), sizeof(req.Account));
    strncpy(req.Password, m_Password.c_str(), sizeof(req.Password));

    m_pMdApi->Login(&req);
    m_Logger->Log->info("ReqUserLogin sent");
}

// ========== 10. 订阅行情 ==========
void NewBrokerMarketGateWay::SubscribeMarketData() {
    std::vector<std::string> tickers;
    for (auto& prop : m_TickerPropertyList) {
        tickers.push_back(prop.Ticker);
    }

    m_pMdApi->Subscribe(tickers);
    m_Logger->Log->info("Subscribed {} tickers", tickers.size());
}

// ========== 11. 数据转换 ==========
void NewBrokerMarketGateWay::ConvertMarketData(NewBrokerMarketData* pData,
                                                MarketData::TFutureMarketData& out) {
    out.Tick = m_TickCount++;

    // 合约信息
    strncpy(out.Ticker, pData->InstrumentID, sizeof(out.Ticker));
    strncpy(out.ExchangeID, pData->ExchangeID, sizeof(out.ExchangeID));

    // 时间信息
    strncpy(out.UpdateTime, pData->UpdateTime, sizeof(out.UpdateTime));
    out.MillSec = pData->MilliSec;

    // 价格信息
    out.LastPrice = pData->LastPrice;
    out.OpenPrice = pData->OpenPrice;
    out.HighestPrice = pData->HighPrice;
    out.LowestPrice = pData->LowPrice;

    // 五档行情
    out.BidPrice1 = pData->BidPrice[0];
    out.BidVolume1 = pData->BidVolume[0];
    out.AskPrice1 = pData->AskPrice[0];
    out.AskVolume1 = pData->AskVolume[0];

    // ... 其他档位

    // 成交信息
    out.Volume = pData->Volume;
    out.Turnover = pData->Turnover;
    out.OpenInterest = pData->OpenInterest;

    // 时间戳
    out.RecvMarketTime = Utils::getTimeUs();
}

// ========== 12. 导出函数 ==========
extern "C" {
    MarketGateWay* CreatePlugin() {
        return new NewBrokerMarketGateWay();
    }
}
```

### 步骤 3: 添加配置加载

**Utils/YMLConfig.hpp** - 添加配置结构：

```cpp
struct NewBrokerConfig {
    std::string ServerAddr;
    std::string Account;
    std::string Password;
    std::string TickerListPath;
};

static bool LoadNewBrokerConfig(const char* yml,
                                NewBrokerConfig& ret,
                                std::string& out) {
    try {
        YAML::Node config = YAML::LoadFile(yml);
        YAML::Node brokerConfig = config["NewBrokerMarketSource"];

        ret.ServerAddr = brokerConfig["ServerAddr"].as<std::string>();
        ret.Account = brokerConfig["Account"].as<std::string>();
        ret.Password = brokerConfig["Password"].as<std::string>();
        ret.TickerListPath = brokerConfig["TickerListPath"].as<std::string>();

        return true;
    } catch (YAML::Exception& e) {
        out = e.what();
        return false;
    }
}
```

### 步骤 4: 创建配置文件

**NewBroker.yml**:

```yaml
NewBrokerMarketSource:
  ServerAddr: tcp://192.168.1.100:8888
  Account: your_account
  Password: your_password
  TickerListPath: ./Config/TickerList.yml
```

### 步骤 5: 编译插件

**CMakeLists.txt** (XMarketCenter/MarketAPI/):

```cmake
# 添加 NewBroker 插件
add_library(NewBrokerMarketGateWay SHARED
    NewBroker/NewBrokerMarketGateWay.cpp
)

# 链接柜台 API
target_link_libraries(NewBrokerMarketGateWay
    newbroker_mdapi  # 柜台提供的库
)

# 设置输出目录
set_target_properties(NewBrokerMarketGateWay PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/build/
)
```

**编译**:

```bash
cd QuantFabric/build
cmake ..
make NewBrokerMarketGateWay -j8

# 输出: build/libNewBrokerMarketGateWay.so
```

### 步骤 6: 测试插件

**运行**:

```bash
./XMarketCenter_0.9.3 XMarketCenter.yml ./libNewBrokerMarketGateWay.so
```

**验证日志**:

```
[INFO] LoadMarketGateWay ./libNewBrokerMarketGateWay.so success
[INFO] Plugin info: CommitID=abc123, UtilsCommitID=def456, APIVersion=1.0.0
[INFO] NewBrokerMarketGateWay starting...
[INFO] API created, version=1.0.0
[INFO] OnConnect: Connected to tcp://192.168.1.100:8888
[INFO] ReqUserLogin sent
[INFO] OnLogin success
[INFO] Subscribed 10 tickers
```

## 插件开发最佳实践

### 1. 错误处理

```cpp
// 柜台 API 调用可能失败，需要检查返回值
int ret = m_pMdApi->Subscribe(tickers);
if (ret != 0) {
    m_Logger->Log->error("Subscribe failed: ret={}", ret);
    // 可能需要重试或报警
}

// 回调中的异常处理
void OnMarketData(NewBrokerMarketData* pData) {
    try {
        // 处理逻辑
        ConvertMarketData(pData, message.FutureMarketData);
        m_MarketMessageQueue.Push(message);
    } catch (const std::exception& e) {
        m_Logger->Log->error("OnMarketData exception: {}", e.what());
    }
}
```

### 2. 性能优化

```cpp
// 避免频繁的字符串操作
// 错误示例：
std::string ticker = pData->InstrumentID;
if (m_TickerMap.find(ticker) != m_TickerMap.end()) { ... }

// 正确示例：
if (m_TickerMap.find(pData->InstrumentID) != m_TickerMap.end()) { ... }

// 预分配内存
m_TickerMap.reserve(1000);  // 预分配 1000 个 Ticker 的空间

// 使用 unordered_map 而非 map
std::unordered_map<std::string, std::string> m_TickerExchangeMap;  // O(1)
// 而非：
std::map<std::string, std::string> m_TickerExchangeMap;  // O(log n)
```

### 3. 日志级别

```cpp
// 关键事件：INFO
m_Logger->Log->info("OnLogin success");

// 调试信息：DEBUG（生产环境关闭）
m_Logger->Log->debug("Recv market: {} LastPrice={}", ticker, price);

// 警告：WARN
m_Logger->Log->warn("OnDisconnect: reason={}", reason);

// 错误：ERROR
m_Logger->Log->error("Subscribe failed: ret={}", ret);
```

### 4. 内存管理

```cpp
// RAII 原则：资源获取即初始化
class NewBrokerMarketGateWay {
public:
    ~NewBrokerMarketGateWay() {
        // 析构函数中释放资源
        if (m_pMdApi) {
            m_pMdApi->Release();
            m_pMdApi = nullptr;
        }
    }
};

// 避免内存泄漏
char** tickers = new char*[count];
// ... 使用 tickers
delete[] tickers;  // 记得释放
```

### 5. 线程安全

```cpp
// 柜台 API 回调通常在独立线程
// 写入无锁队列是线程安全的（SPSC）
void OnMarketData(NewBrokerMarketData* pData) {
    // 安全：无锁队列写入
    m_MarketMessageQueue.Push(message);

    // 不安全：访问共享变量
    // m_SharedData++;  // 错误！需要加锁或原子操作
}

// 如果确实需要共享变量，使用原子操作
std::atomic<int> m_SharedCount{0};
void OnMarketData(NewBrokerMarketData* pData) {
    m_SharedCount.fetch_add(1, std::memory_order_relaxed);
}
```

## 常见问题

### Q1: 插件加载失败

**错误**: `Symbol 'CreatePlugin' not found`

**原因**: 忘记导出函数或函数签名不匹配

**解决**:

```cpp
// 确保有 extern "C" 和正确的函数签名
extern "C" {
    MarketGateWay* CreatePlugin() {
        return new NewBrokerMarketGateWay();
    }
}

// 检查符号是否存在
nm -D libNewBrokerMarketGateWay.so | grep CreatePlugin
```

### Q2: 柜台 API 库找不到

**错误**: `error while loading shared libraries: libnewbroker.so: cannot open shared object file`

**解决**:

```bash
# 1. 将柜台 API 库路径添加到 LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/path/to/broker/lib:$LD_LIBRARY_PATH

# 2. 或者将库拷贝到系统路径
sudo cp libnewbroker.so /usr/local/lib/
sudo ldconfig

# 3. 或者使用 rpath
cmake -DCMAKE_BUILD_RPATH=/path/to/broker/lib ..
```

### Q3: 数据转换字段不匹配

**问题**: 柜台字段名与 TFutureMarketData 不一致

**解决**:

```cpp
// 建立映射关系
void ConvertMarketData(NewBrokerMarketData* pData,
                      MarketData::TFutureMarketData& out) {
    // 柜台字段 -> 标准字段
    out.LastPrice = pData->last_price;      // 下划线命名
    out.BidPrice1 = pData->bid[0].price;    // 数组形式
    out.BidVolume1 = pData->bid[0].volume;

    // 处理特殊情况
    if (pData->last_price <= 0) {
        out.LastPrice = out.PreClosePrice;  // 使用昨收价
    }
}
```

### Q4: 性能不达标

**问题**: 延迟超过预期

**排查**:

1. **检查日志级别**: DEBUG 日志会严重影响性能

```cpp
// 生产环境使用 INFO 或 WARN
m_Logger->Log->set_level(spdlog::level::info);
```

2. **检查 CPU 绑定**:

```bash
# 绑定到隔离 CPU
taskset -c 5 ./XMarketCenter_0.9.3 ...
```

3. **检查队列容量**:

```cpp
// 队列太小会频繁满，导致数据丢失
LockFreeQueue<Message::PackMessage> m_MarketMessageQueue(1 << 14);  // 16K
```

4. **使用性能分析工具**:

```bash
# perf 分析热点
perf record -g ./XMarketCenter_0.9.3 ...
perf report

# 查看 CPU 使用率
top -H -p $(pgrep XMarketCenter)
```

## 总结

插件开发的关键点：

1. **实现接口**: 继承 `MarketGateWay`，实现 4 个虚函数
2. **回调处理**: 实现柜台 API 的回调接口
3. **数据转换**: 将柜台数据转换为标准格式
4. **无锁队列**: 使用 `m_MarketMessageQueue` 传递数据
5. **导出函数**: `extern "C" { MarketGateWay* CreatePlugin() }`

遵循最佳实践，可以开发出高性能、高可靠的行情插件。

**下一步**: 阅读 [性能调优指南](04_PerformanceTuning.md) 深入优化性能。
