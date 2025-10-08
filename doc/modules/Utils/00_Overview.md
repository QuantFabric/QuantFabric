# Utils 模块文档

## 模块概述

Utils 是 QuantFabric 交易系统的**基础工具库**，提供所有交易组件共用的底层功能，包括配置管理、网络通信、进程间通信(IPC)、日志系统、数据结构等核心组件。

**项目地址**: [Utils](https://github.com/QuantFabric/Utils)

## 核心组件

### 1. 配置管理

#### YMLConfig.hpp
提供基于 YAML-CPP 的配置文件加载功能。

**主要配置结构**:
- `CTPMarketSourceConfig`: CTP 行情源配置
- `REMMarketSourceConfig`: 盛立 REM 行情源配置
- `RawMarketSourceConfig`: 原始行情源配置
- `XServerConfig`: XServer 服务端配置
- `XWatcherConfig`: XWatcher 监控配置
- `XTraderConfig`: XTrader 交易网关配置
- `XRiskJudgeConfig`: XRiskJudge 风控配置
- `XQuantConfig`: XQuant 策略配置

**使用示例**:
```cpp
#include "YMLConfig.hpp"

CTPMarketSourceConfig config;
std::string errorMsg;
if (Utils::LoadCTPMarkeSourceConfig("config.yml", config, errorMsg)) {
    // 使用配置
}
```

### 2. 网络通信

#### HPPackClient (客户端)
基于 HPSocket 框架的网络客户端，支持 Pack 协议。

**核心方法**:
- `Start(ip, port)`: 启动客户端连接
- `SendData(buffer, length)`: 发送数据
- `RegisterLoginCallBack()`: 注册登录回调
- `RegisterMessageCallBack()`: 注册消息回调

**特点**:
- 自动重连机制
- 心跳保活
- 二进制 Pack 协议

#### HPPackServer (服务端)
基于 HPSocket 框架的网络服务端。

**核心方法**:
- `Start(ip, port)`: 启动服务端监听
- `SendData(connId, buffer, length)`: 向指定连接发送数据
- `Disconnect(connId)`: 断开指定连接

### 3. 进程间通信 (IPC)

#### IPCLockFreeQueue.hpp
**无锁队列**，用于高性能进程间通信。

**技术特点**:
- 基于共享内存实现
- SPSC (Single Producer Single Consumer) 模式
- 无锁设计，适合高频交易场景

**使用场景**:
- XQuant → XTrader: 报单请求
- XTrader → XQuant: 订单回报、仓位、资金信息
- XRiskJudge ↔ XTrader: 风控检查

**示例**:
```cpp
#include "IPCLockFreeQueue.hpp"

// 发送端
IPCLockFreeQueue queue("OrderQueue", 1024*1024, true);
OrderRequest req;
queue.Push(&req, sizeof(OrderRequest));

// 接收端
IPCLockFreeQueue queue("OrderQueue", 1024*1024, false);
OrderRequest req;
if (queue.Pop(&req, sizeof(OrderRequest))) {
    // 处理订单
}
```

#### IPCMarketQueue.hpp
**行情专用队列**，优化的行情数据传输。

**特点**:
- 针对行情数据结构优化
- 高吞吐量设计
- 支持多订阅者模式

**使用场景**:
- XMarketCenter → XQuant: 行情推送
- XMarketCenter → HFTrader: 行情推送

### 4. 内存数据结构

#### LockFreeQueue.hpp
**单进程内无锁队列**。

**特点**:
- 基于 CAS (Compare-And-Swap) 实现
- 线程安全
- 高性能

#### RingBuffer.hpp
**环形缓冲区**。

**特点**:
- 固定大小
- 覆盖写入模式
- 适合日志缓存、性能监控

### 5. 日志系统

#### Logger.h/cpp
封装 SPDLog/FMTLog 的日志工具。

**特点**:
- 异步日志
- 文件轮转
- 多级别日志 (DEBUG, INFO, WARN, ERROR)

**使用**:
```cpp
#include "Logger.h"

Logger::Init("./logs/app.log");
Logger::Info("Application started");
Logger::Error("Error: {}", errorMsg);
```

### 6. 消息协议

#### PackMessage.hpp
定义系统所有消息类型和数据结构。

**消息类型**:
```cpp
#define MESSAGE_FUTUREMARKET       "FutureMarket"      // 期货行情
#define MESSAGE_STOCKMARKET        "StockMarket"       // 股票行情
#define MESSAGE_ORDERSTATUS        "OrderStatus"       // 订单状态
#define MESSAGE_ACCOUNTFUND        "AccountFund"       // 账户资金
#define MESSAGE_ACCOUNTPOSITION    "AccountPosition"   // 账户仓位
#define MESSAGE_EVENTLOG           "EventLog"          // 事件日志
#define MESSAGE_RISKREPORT         "RiskReport"        // 风控报告
```

**关键结构**:
- `TLoginRequest/TLoginResponse`: 登录请求/响应
- `TOrderRequest`: 报单请求
- `TActionRequest`: 撤单请求
- `TOrderStatus`: 订单状态
- `TAccountFund`: 账户资金
- `TAccountPosition`: 账户仓位
- `TRiskReport`: 风控报告

**客户端类型**:
```cpp
enum EClientType {
    EXTRADER = 1,           // XTrader 交易网关
    EXMONITOR = 2,          // XMonitor GUI 客户端
    EXMARKETCENTER = 3,     // XMarketCenter 行情网关
    EXRISKJUDGE = 4,        // XRiskJudge 风控系统
    EXWATCHER = 5,          // XWatcher 监控组件
    EXQUANT = 6,            // XQuant 策略引擎
    EHFTRADER = 7,          // HFTrader 高频交易
    EXDATAPLAYER = 8,       // XDataPlayer 数据播放
};
```

### 7. 数据库操作

#### SQLiteManager.hpp
SQLite 数据库封装。

**功能**:
- 连接管理
- SQL 执行
- 事务支持
- 结果集处理

**使用场景**:
- XServer 用户权限管理
- XTrader 订单持久化
- 历史数据存储

### 8. 工具函数

#### Util.hpp
通用工具函数集合。

**时间函数**:
```cpp
unsigned long getTimeNs();          // 纳秒时间戳
unsigned long getTimeUs();          // 微秒时间戳
unsigned long getTimeMs();          // 毫秒时间戳
unsigned long getTimeSec();         // 秒级时间戳
const char* getCurrentTimeSec();    // 格式化时间字符串
```

**文件操作**:
- `createDirectory()`: 创建目录
- `pathExists()`: 检查路径存在
- `removeFile()`: 删除文件

**字符串处理**:
- `split()`: 字符串分割
- `trim()`: 去除空白
- `replace()`: 字符串替换
- `GBKToUTF8()`: 编码转换

**CPU 绑定**:
```cpp
int BindCPU(int cpuID);  // 绑定当前线程到指定 CPU
```

### 9. 其他组件

#### SnapShotHelper.hpp
**快照工具**，用于性能监控和状态保存。

#### XPluginEngine.hpp
**插件引擎**，用于 XMonitor 的插件架构。

#### MarketData.hpp
**行情数据结构**定义。

```cpp
struct FutureMarketData {
    char Ticker[32];
    char ExchangeID[16];
    double LastPrice;
    double Volume;
    double BidPrice[5];
    double AskPrice[5];
    int BidVolume[5];
    int AskVolume[5];
    // ... 更多字段
};
```

## 编译说明

Utils 是**头文件库**，大部分功能以 `.hpp` 形式提供，无需单独编译。

部分 `.cpp` 文件（如 HPPackClient.cpp、Logger.cpp）需要与使用它的组件一起编译。

## 依赖项

### 必需
- **YAML-CPP**: 配置文件解析 (XAPI/YAML-CPP/0.8.0)
- **HPSocket**: 网络通信框架 (XAPI/HP-Socket/5.8.2)
- **SPDLog**: 日志库 (XAPI/SPDLog)
- **parallel_hashmap**: 高性能哈希表 (XAPI/parallel_hashmap)

### 可选
- **SQLite3**: 数据库功能 (`yum install sqlite-devel`)

## 测试用例

Utils 提供了完整的测试用例（`*Test.cpp`）：

- `YMLConfigTest.cpp`: 配置加载测试
- `HPPackClientTest.cpp`: 网络客户端测试
- `HPPackServerTest.cpp`: 网络服务端测试
- `LockFreeQueueTest.cpp`: 无锁队列测试
- `IPCLockFreeQueueTest.cpp`: IPC 队列测试
- `IPCMarketQueueTest.cpp`: 行情队列测试
- `RingBufferTest.cpp`: 环形缓冲区测试
- `LoggerTest.cpp`: 日志系统测试
- `SQLiteManagerTest.cpp`: 数据库测试
- `SnapShotHelperTest.cpp`: 快照工具测试

**编译测试**:
```bash
g++ -std=c++17 LockFreeQueueTest.cpp -o test -lpthread
./test
```

## 性能优化要点

1. **无锁设计**: IPCLockFreeQueue 和 LockFreeQueue 采用无锁算法，避免互斥锁开销
2. **内存对齐**: 关键数据结构使用 `alignas()` 进行缓存行对齐
3. **零拷贝**: 共享内存 IPC 避免数据拷贝
4. **预分配**: 队列在初始化时预分配内存
5. **SPSC 优化**: 单生产者单消费者队列针对性优化

## 注意事项

### ABI 兼容性
所有使用 Utils 的组件必须使用相同的 ABI 设置：
```cmake
add_definitions(-D_GLIBCXX_USE_CXX11_ABI=0)
```

### 共享内存权限
IPCLockFreeQueue 需要足够的共享内存权限：
```bash
# 查看共享内存限制
ipcs -l

# 增加共享内存大小（临时）
sysctl -w kernel.shmmax=17179869184
```

### CPU 亲和性
使用 `BindCPU()` 时需要：
- Root 权限或 CAP_SYS_NICE 能力
- CPU 隔离配置 (`isolcpus` 内核参数)

## 使用示例

### 完整的消息通信示例

```cpp
// 发送端（XQuant）
#include "HPPackClient.h"
#include "PackMessage.hpp"

HPPackClient client;
client.RegisterLoginCallBack(OnLoginCallback);
client.RegisterMessageCallBack(OnMessageCallback);
client.Start("127.0.0.1", 8001);

// 发送报单请求
Message::TOrderRequest order;
strcpy(order.Ticker, "rb2505");
order.Direction = Message::EBUY;
order.OrderPrice = 3500.0;
order.OrderVolume = 1;
client.SendData((unsigned char*)&order, sizeof(order));

// 接收端（XTrader）
#include "HPPackServer.h"

HPPackServer server;
server.RegisterOnAcceptCallBack(OnAcceptCallback);
server.RegisterOnReceiveCallBack(OnReceiveCallback);
server.Start("0.0.0.0", 8001);
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/Utils
