# XMarketCenter 模块文档

## 模块概述

XMarketCenter 是 QuantFabric 交易系统的**行情网关**，负责从各期货公司/证券公司柜台接收行情数据，并通过共享内存和网络将行情分发给策略程序和监控系统。

**项目地址**: [XMarketCenter](https://github.com/QuantFabric/XMarketCenter)

## 核心功能

1. **接收行情数据**: 从 CTP、REM、YD 等柜台 API 接收实时行情
2. **共享内存写入**: 将行情写入共享内存队列，供 XQuant/HFTrader 读取
3. **网络转发**: 通过 HPSocket 将行情转发至 XWatcher 监控组件
4. **插件架构**: 通过动态加载插件适配不同柜台 API

## 架构设计

### 整体架构图

```
柜台 API (CTP/REM/YD)
         ↓
  MarketGateWay 插件
         ↓
    ┌─────────┴─────────┐
    ↓                   ↓
共享内存队列        HPPackClient
    ↓                   ↓
XQuant/HFTrader      XWatcher → XServer → XMonitor
```

### 插件架构

XMarketCenter 使用**插件模式**，通过 `dlopen()` 动态加载不同柜台的行情网关：

```cpp
// 加载插件
MarketGateWay* m_MarketGateWay = XPluginEngine<MarketGateWay>::LoadPlugin(soPath, errorString);

// 运行插件
m_MarketGateWay->Run();
```

### 线程模型

XMarketCenter 采用**单线程模型**：
- 主线程：加载配置、初始化组件、启动插件
- 插件内线程：由柜台 API 回调驱动 (异步)

## 核心组件

### 1. XMarketCenter (主程序)

**文件**: `XMarketCenter.cpp`、`XMarketCenter.h`

**主要职责**:
- 加载配置文件 (`XMarketCenter.yml`)
- 加载 Ticker 列表 (`TickerList.yml`)
- 动态加载 MarketGateWay 插件
- 初始化网络客户端 (HPPackClient)
- 初始化发布服务器 (PubServer)

**启动流程**:
```cpp
1. LoadConfig()           // 加载配置
2. LoadMarketGateWay()    // 加载插件
3. Run()                  // 启动主逻辑
   - 启动 HPPackClient
   - 登录到 XWatcher
   - 启动插件
   - 循环读取共享内存队列
   - 转发行情到 XWatcher
```

### 2. MarketGateWay (插件基类)

**文件**: `MarketAPI/MarketGateWay.hpp`

**抽象接口**:
```cpp
class MarketGateWay {
public:
    virtual bool LoadAPIConfig() = 0;           // 加载柜台 API 配置
    virtual void Run() = 0;                     // 启动行情接收
    virtual void GetCommitID(...) = 0;          // 获取版本信息
    virtual void GetAPIVersion(...) = 0;        // 获取 API 版本
protected:
    Utils::MarketCenterConfig m_MarketCenterConfig;  // 配置
    LockFreeQueue<Message::PackMessage> m_MarketMessageQueue;  // 行情队列
    Utils::Logger* m_Logger;                    // 日志
};
```

**关键成员**:
- `m_MarketMessageQueue`: **无锁队列**，存储行情消息
- `m_TickerPropertyMap`: Ticker 属性映射表 (合约代码 → 属性)
- `m_TickerExchangeMap`: Ticker 交易所映射表

### 3. HPPackClient (网络客户端)

**文件**: `HPPackClient.cpp`、`HPPackClient.h`

**功能**:
- 连接到 XWatcher 监控组件
- 发送行情数据
- 发送事件日志

**关键方法**:
```cpp
void Start();                           // 启动客户端
void Login(const char* exchangeID);     // 登录
void SendData(unsigned char* buffer, int len);  // 发送数据
```

### 4. PubServer (发布服务器)

**文件**: `PubServer.hpp`

**功能**:
- 基于 SHMServer 的发布-订阅模式
- 将行情发布到共享内存
- 供 XQuant/HFTrader 订阅

**关键方法**:
```cpp
void Publish(const char* topic, const void* data, size_t len);
```

### 5. MarketReader (行情读取器)

**文件**: `MarketReader/MarketReader.hpp`

**功能**:
- 从共享内存队列读取行情
- 供测试和调试使用

## 插件实现

### CTP 插件

**文件**: `MarketAPI/CTP/CTPMarketGateWay.cpp`

**实现细节**:
```cpp
class CTPMarketGateWay : public MarketGateWay, public CThostFtdcMdSpi {
public:
    bool LoadAPIConfig() override;      // 加载 CTP 配置
    void Run() override;                // 启动 CTP 行情

    // CTP 回调函数
    void OnFrontConnected() override;
    void OnRspUserLogin(...) override;
    void OnRtnDepthMarketData(...) override;  // 行情回调
};
```

**行情接收流程**:
1. 连接柜台前置: `pMdUserApi->RegisterFront()`
2. 登录: `pMdUserApi->ReqUserLogin()`
3. 订阅合约: `pMdUserApi->SubscribeMarketData()`
4. 接收行情: `OnRtnDepthMarketData()` 回调
5. 数据转换: CTP 结构 → `TFutureMarketData`
6. 写入队列: `m_MarketMessageQueue.Push()`

**配置示例** (`CTP.yml`):
```yaml
CTPMarketSource:
  FrontAddr: tcp://180.168.146.187:10131
  BrokerID: 9999
  Account: 123456
  Password: ******
  TickerListPath: /path/to/TickerList.yml
```

### REM 插件

**文件**: `MarketAPI/REM/REMMarketGateWay.cpp`

**特点**:
- 支持 **UDP 多播行情** (低延迟)
- 支持 TCP 行情 (备用)

**配置示例** (`REM.yml`):
```yaml
REMMarketSource:
  Account: user001
  Password: ******
  FrontIP: 192.168.1.100
  FrontPort: 5555
  MultiCastIP: 239.0.0.1    # 多播地址
  MultiCastPort: 6666
  LocalIP: 192.168.1.10     # 本机 IP
  LocalPort: 7777
  TickerListPath: /path/to/TickerList.yml
```

### Test 插件

**文件**: `MarketAPI/Test/TestMarketGateWay.cpp`

**用途**:
- 模拟行情数据
- 用于测试和开发

**功能**:
- 从配置文件读取模拟价格
- 定时生成行情 Tick
- 无需连接真实柜台

### XueQiu (雪球) 插件

**文件**: `MarketAPI/XueQiu/XueQiuMarketGateWay.cpp`

**用途**:
- 获取股票行情数据
- 通过雪球 API 获取实时价格

## 数据结构

### 行情数据 (TFutureMarketData)

```cpp
struct TFutureMarketData {
    unsigned int Tick;              // Tick 序号
    char Ticker[32];                // 合约代码
    char ExchangeID[16];            // 交易所代码
    char UpdateTime[16];            // 更新时间 "09:30:00"
    int MillSec;                    // 毫秒
    double LastPrice;               // 最新价
    double Volume;                  // 成交量
    double Turnover;                // 成交额
    double OpenInterest;            // 持仓量
    double OpenPrice;               // 开盘价
    double HighestPrice;            // 最高价
    double LowestPrice;             // 最低价
    double PreSettlementPrice;      // 昨结算
    double PreClosePrice;           // 昨收价
    double BidPrice1;               // 买一价
    double BidPrice2;               // 买二价
    // ... 买三、买四、买五
    double AskPrice1;               // 卖一价
    double AskPrice2;               // 卖二价
    // ... 卖三、卖四、卖五
    int BidVolume1;                 // 买一量
    // ... 买二量至买五量
    int AskVolume1;                 // 卖一量
    // ... 卖二量至卖五量
};
```

### Ticker 配置 (TickerList.yml)

```yaml
TickerList:
  - Ticker: rb2505
    ExchangeID: SHFE
    ProductID: rb
    Index: 0
  - Ticker: au2506
    ExchangeID: SHFE
    ProductID: au
    Index: 1
  - Ticker: IF2501
    ExchangeID: CFFEX
    ProductID: IF
    Index: 2
```

## 配置文件

### 主配置 (XMarketCenter.yml)

```yaml
XMarketCenterConfig:
  ExchangeID: SHFE               # 交易所 ID
  ServerIP: 127.0.0.1            # XWatcher IP
  Port: 8001                     # XWatcher 端口
  TotalTick: 10000               # 总 Tick 数
  MarketServer: MarketServer     # 共享内存名称
  RecvTimeOut: 300               # 接收超时 (秒)
  APIConfig: ./Config/CTP.yml    # 柜台 API 配置文件
  TickerListPath: ./Config/TickerList.yml  # Ticker 列表
  ToMonitor: true                # 是否转发到监控
  Future: true                   # 是否期货 (false=股票)
  CPUSET: 5                      # CPU 绑定
```

### CTP 柜台配置 (CTP.yml)

```yaml
CTPMarketSource:
  FrontAddr: tcp://180.168.146.187:10131  # 柜台前置地址
  BrokerID: 9999                          # 期货公司代码
  Account: 123456                         # 账户
  Password: password123                   # 密码
  TickerListPath: ./Config/TickerList.yml # Ticker 列表
```

## 编译构建

### CMakeLists.txt 配置

```cmake
cmake_minimum_required(VERSION 3.10)
PROJECT(XMarketCenter)

set(CMAKE_CXX_STANDARD 17)
add_definitions(-D_GLIBCXX_USE_CXX11_ABI=0)

# 依赖库
include_directories(../Utils/)
include_directories(../XAPI/HP-Socket/5.8.2/include/)
include_directories(../XAPI/YAML-CPP/0.8.0/include/)
include_directories(../XAPI/FMTLogger/include/)

link_directories(../XAPI/HP-Socket/5.8.2/lib)
link_directories(../XAPI/YAML-CPP/0.8.0/lib)

# 主程序
add_executable(XMarketCenter_0.9.3
    main.cpp
    XMarketCenter.cpp
    HPPackClient.cpp
)

target_link_libraries(XMarketCenter_0.9.3
    pthread
    hpsocket4c
    yaml-cpp
    dl        # 动态加载插件
)

# 插件编译
add_subdirectory(MarketAPI)
```

### 插件编译 (MarketAPI/CMakeLists.txt)

```cmake
# CTP 插件
add_library(CTPMarketGateWay SHARED CTP/CTPMarketGateWay.cpp)
target_link_libraries(CTPMarketGateWay thostmduserapi_se)

# REM 插件
add_library(REMMarketGateWay SHARED REM/REMMarketGateWay.cpp)
target_link_libraries(REMMarketGateWay EesMarketApi)

# Test 插件
add_library(TestMarketGateWay SHARED Test/TestMarketGateWay.cpp)
```

### 编译命令

```bash
cd QuantFabric
sh build_release.sh
# 或
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j8
```

**输出文件**:
- `build/XMarketCenter_0.9.3`: 主程序
- `build/libCTPMarketGateWay.so`: CTP 插件
- `build/libREMMarketGateWay.so`: REM 插件
- `build/libTestMarketGateWay.so`: 测试插件

## 运行方式

### 命令行启动

```bash
# 基本启动
./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so

# CPU 绑定启动
taskset -c 5 ./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so

# 使用 Onload 加速
onload ./XMarketCenter_0.9.3 XMarketCenter.yml libCTPMarketGateWay.so
```

### 启动参数

```cpp
int main(int argc, char* argv[]) {
    // argv[1]: 配置文件路径
    // argv[2]: 插件 .so 文件路径
    XMarketCenter marketCenter;
    marketCenter.LoadConfig(argv[1]);
    marketCenter.LoadMarketGateWay(argv[2]);
    marketCenter.Run();
    return 0;
}
```

### 生产环境部署

1. **Colo 机房部署**: 靠近柜台机房，降低网络延迟
2. **CPU 绑定**: 绑定到隔离 CPU，避免调度开销
3. **网络优化**: 使用 Onload 或 TCPDirect
4. **日志配置**: 调整日志级别为 INFO/WARN，减少 I/O

**systemd 服务配置** (`xmarketcenter.service`):
```ini
[Unit]
Description=XMarketCenter Market Gateway
After=network.target

[Service]
Type=simple
User=trader
WorkingDirectory=/home/trader/QuantFabric/build
ExecStart=/usr/bin/taskset -c 5 /home/trader/QuantFabric/build/XMarketCenter_0.9.3 ../XMarketCenter/XMarketCenter.yml ./libCTPMarketGateWay.so
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 数据流向

### 行情接收流程

```
1. 柜台 API 回调
   OnRtnDepthMarketData()

2. 数据转换
   CTP 结构 → TFutureMarketData

3. 写入无锁队列
   m_MarketMessageQueue.Push()

4. 主线程读取队列
   while (m_MarketMessageQueue.Pop())

5. 并行处理
   ├─ 写入共享内存 → XQuant/HFTrader
   └─ 发送网络 → XWatcher → XMonitor
```

### 共享内存发布

```cpp
// PubServer 发布行情
m_PubServer->Publish("MarketServer", &marketData, sizeof(TFutureMarketData));

// XQuant 订阅行情
SHMConnection conn("MarketServer");
TFutureMarketData data;
if (conn.Receive(&data, sizeof(data))) {
    // 处理行情
}
```

## 性能优化

### 1. 无锁队列
- 使用 `LockFreeQueue` 避免锁竞争
- SPSC 模式优化

### 2. CPU 绑定
```cpp
Utils::BindCPU(m_MarketCenterConfig.CPUSET);  // 绑定到指定 CPU
```

### 3. 共享内存
- 零拷贝数据传输
- 直接内存映射

### 4. 网络优化
- 使用 Solarflare Onload
- 减少系统调用

### 5. 日志优化
- 异步日志
- 关键路径避免日志输出

## 监控与调试

### 日志输出

```bash
# 日志位置
./logs/XMarketCenter_YYYYMMDD.log

# 日志内容
[2025-01-15 09:30:00.123] [info] XMarketCenter::LoadConfig successed
[2025-01-15 09:30:00.234] [info] LoadMarketGateWay libCTPMarketGateWay.so successed
[2025-01-15 09:30:05.456] [info] OnRtnDepthMarketData: rb2505 LastPrice=3500.0
```

### 性能监控

```cpp
// 在代码中添加性能计数
unsigned long startTime = Utils::getTimeUs();
// ... 处理逻辑
unsigned long endTime = Utils::getTimeUs();
printf("Process latency: %lu us\n", endTime - startTime);
```

### 常见问题排查

**问题 1**: 连接柜台失败
```bash
# 检查网络连通性
ping 180.168.146.187
telnet 180.168.146.187 10131

# 检查配置文件
cat Config/CTP.yml
```

**问题 2**: 收不到行情
```bash
# 检查订阅列表
cat Config/TickerList.yml

# 检查日志
tail -f logs/XMarketCenter_*.log | grep "OnRtnDepthMarketData"
```

**问题 3**: 共享内存写入失败
```bash
# 检查共享内存
ipcs -m | grep MarketServer

# 清理共享内存
ipcrm -M $(ipcs -m | grep MarketServer | awk '{print $1}')
```

## 扩展开发

### 添加新柜台插件

1. **创建插件目录**:
```bash
mkdir XMarketCenter/MarketAPI/NewBroker
```

2. **实现 MarketGateWay 接口**:
```cpp
// NewBrokerMarketGateWay.h
class NewBrokerMarketGateWay : public MarketGateWay {
public:
    bool LoadAPIConfig() override;
    void Run() override;
    // ... 其他方法
};
```

3. **实现柜台回调**:
```cpp
// 继承柜台 API 回调类
class NewBrokerMarketGateWay : public NewBrokerMdSpi {
    void OnMarketData(NewBrokerMarketData* pData) {
        // 转换为 TFutureMarketData
        Message::PackMessage msg;
        // ... 填充数据
        m_MarketMessageQueue.Push(msg);
    }
};
```

4. **添加到 CMakeLists.txt**:
```cmake
add_library(NewBrokerMarketGateWay SHARED
    NewBroker/NewBrokerMarketGateWay.cpp
)
target_link_libraries(NewBrokerMarketGateWay newbrokerapi)
```

5. **编译使用**:
```bash
make
./XMarketCenter_0.9.3 XMarketCenter.yml ./libNewBrokerMarketGateWay.so
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XMarketCenter
**QQ 群**: 748930268 (验证码: QuantFabric)
