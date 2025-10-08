# XAPI 模块文档

## 模块概述

XAPI 是 QuantFabric 开发所依赖的**第三方库集合**，包括基础工具库、网络通信框架、以及各券商柜台 API。

**项目地址**: [XAPI](https://github.com/QuantFabric/XAPI)

## 目录结构

```
XAPI/
├── YAML-CPP/           # YAML 配置文件解析库
├── SPDLog/             # 高性能日志库
├── FMTLogger/          # FMT 格式化日志库
├── HP-Socket/          # 高性能网络通信框架
├── ConcurrentQueue/    # 并发队列库
├── CPP-IPC/            # C++ 进程间通信库
├── parallel_hashmap/   # 并行哈希表
├── CTP/                # 上期所 CTP 柜台 API (期货)
├── REM/                # 盛立 REM 柜台 API (期货)
├── YD/                 # 易达 YD 柜台 API (期货)
├── OES/                # 宽睿 OES 柜台 API (股票)
├── XTP/                # 中泰 XTP 柜台 API (股票)
├── Tora/               # 华鑫 Tora 柜台 API (股票)
├── onload-7.1.3.202/   # Solarflare 网卡驱动
├── onload-8.0.0.34/    # Solarflare 网卡驱动 (新版本)
└── tcpdirect-8.0.1.2/  # TCPDirect 内核旁路库
```

## 基础库

### 1. YAML-CPP (0.8.0)

**用途**: YAML 配置文件解析

**位置**: `XAPI/YAML-CPP/0.8.0/`

**目录结构**:
```
YAML-CPP/0.8.0/
├── include/yaml-cpp/   # 头文件
└── lib/                # 库文件
    └── libyaml-cpp.a   # 静态库
```

**在 QuantFabric 中的使用**:
- 所有组件的配置文件加载 (通过 Utils/YMLConfig.hpp)
- XTrader、XMarketCenter、XQuant 等配置解析

**编译注意事项** (Ubuntu):
```bash
cd XAPI/YAML-CPP/0.8.0/
mkdir build && cd build
cmake .. -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=0"
make
```
**关键**: 必须使用 `-D_GLIBCXX_USE_CXX11_ABI=0` 确保 ABI 兼容性

### 2. SPDLog

**用途**: 高性能异步日志库

**位置**: `XAPI/SPDLog/`

**特点**:
- 头文件库，无需编译
- 异步日志输出
- 支持多种日志级别
- 文件轮转

**在 QuantFabric 中的使用**:
- Utils/Logger 封装 SPDLog
- 所有组件的日志输出

### 3. FMTLogger

**用途**: 基于 FMT 库的格式化日志

**位置**: `XAPI/FMTLogger/`

**特点**:
- 比 printf 更快的格式化
- 类型安全
- 支持自定义类型格式化

**编译定义**:
```cmake
add_definitions(-DFMT_HEADER_ONLY)
add_definitions(-DFMTLOG_HEADER_ONLY)
```

### 4. HP-Socket (5.8.2)

**用途**: 高性能跨平台网络通信框架

**位置**: `XAPI/HP-Socket/5.8.2/`

**目录结构**:
```
HP-Socket/5.8.2/
├── include/            # C/C++ 头文件
└── lib/
    └── libhpsocket4c.a # C 版本静态库
```

**特点**:
- 支持 TCP/UDP/HTTP/WebSocket
- Pack 协议支持 (QuantFabric 使用)
- 高并发、低延迟

**在 QuantFabric 中的使用**:
- Utils/HPPackClient: 客户端封装
- Utils/HPPackServer: 服务端封装
- XServer、XWatcher、XTrader、XMarketCenter 等组件的网络通信

**CMake 配置**:
```cmake
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/HP-Socket/5.8.2/include/)
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/HP-Socket/5.8.2/lib)
target_link_libraries(${APP_NAME} hpsocket4c)
```

### 5. ConcurrentQueue

**用途**: 高性能并发队列

**位置**: `XAPI/ConcurrentQueue/`

**特点**:
- 无锁设计
- MPMC (Multi-Producer Multi-Consumer)
- 头文件库

**在 QuantFabric 中的使用**:
- 部分组件内部线程间通信
- 作为 IPCLockFreeQueue 的参考实现

### 6. CPP-IPC

**用途**: C++ 进程间通信库

**位置**: `XAPI/CPP-IPC/`

**特点**:
- 共享内存
- 消息队列
- 信号量

**在 QuantFabric 中的使用**:
- 作为 SHMServer 的参考实现

### 7. parallel_hashmap

**用途**: 高性能并行哈希表

**位置**: `XAPI/parallel_hashmap/`

**特点**:
- 比 std::unordered_map 快 2-3 倍
- 内存占用更小
- 线程安全版本

**在 QuantFabric 中的使用**:
- XTrader 订单管理
- XRiskJudge 风控状态管理
- XQuant 策略数据存储

## 期货柜台 API

### 1. CTP (上期所综合交易平台)

**位置**: `XAPI/CTP/`

**支持的交易所**:
- 中金所 (CFFEX): 股指期货、国债期货
- 上期所 (SHFE): 黑色、有色金属
- 大商所 (DCE): 农产品
- 郑商所 (CZCE): 农产品、化工品
- 上海能源中心 (INE): 能源化工

**版本**: 多个版本支持不同柜台
```
CTP/
├── v6.3.15/        # 常用版本
├── v6.6.1/         # 较新版本
└── v6.7.2/         # 最新版本
```

**API 文件**:
- `thostmduserapi.h`: 行情 API
- `thosttraderapi.h`: 交易 API
- `libthostmduserapi_se.so`: 行情动态库
- `libthosttraderapi_se.so`: 交易动态库

**在 QuantFabric 中的使用**:
- `XMarketCenter/MarketAPI/CTP/`: CTP 行情网关插件
- `XTrader/TraderAPI/CTP/`: CTP 交易网关插件

### 2. REM (盛立期货柜台)

**位置**: `XAPI/REM/`

**特点**:
- 多播行情
- TCP 交易
- 低延迟

**API 文件**:
- `EESQuoteApi.h`: 行情 API
- `EESTraderApi.h`: 交易 API

**在 QuantFabric 中的使用**:
- `XMarketCenter/MarketAPI/REM/`: REM 行情网关插件
- `XTrader/TraderAPI/REM/`: REM 交易网关插件

### 3. YD (易达期货柜台)

**位置**: `XAPI/YD/`

**特点**:
- 极低延迟 (纳秒级)
- 支持内核旁路
- 适合高频交易

**版本**: 多个平台版本
```
YD/
├── x86/           # x86 架构
├── x64/           # x64 架构
└── arm64/         # ARM64 架构
```

**API 文件**:
- `ydApi.h`: 统一 API 头文件
- `libydApi.so`: 动态库

**在 QuantFabric 中的使用**:
- `XTrader/TraderAPI/YD/`: YD 交易网关插件
- **HFTrader 商业版**支持 YD 柜台

**性能指标** (HFTrader + YD + 5.0GHz CPU):
- Tick2Order 中位数: **1.1μs**
- 最小延迟: **785ns**

## 股票柜台 API

### 1. OES (宽睿股票柜台)

**位置**: `XAPI/OES/`

**支持的交易所**:
- 上交所 (SSE)
- 深交所 (SZSE)

**特点**:
- 支持股票、债券、基金
- 支持融资融券
- 支持期权

**在 QuantFabric 中的使用**:
- `XTrader/TraderAPI/OES/`: OES 交易网关插件

### 2. XTP (中泰证券柜台)

**位置**: `XAPI/XTP/`

**特点**:
- 支持普通交易、两融、期权
- 多账户管理
- 高性能

**在 QuantFabric 中的使用**:
- **HFTrader 商业版**支持 XTP 柜台
- **StrikeBoarder 打板系统**支持 XTP

### 3. Tora (华鑫证券柜台)

**位置**: `XAPI/Tora/`

**特点**:
- 托管机房低延迟
- 适合高频交易
- 支持做市、套利策略

**在 QuantFabric 中的使用**:
- **HFTrader 商业版**支持 Tora 柜台
- **StrikeBoarder 打板系统**支持 Tora

## 网络优化库

### 1. Solarflare Onload

**用途**: 内核旁路 TCP/UDP 加速

**位置**:
- `XAPI/onload-7.1.3.202/` (旧版本)
- `XAPI/onload-8.0.0.34/` (新版本)

**特点**:
- 内核旁路，直接访问网卡
- 降低延迟 50-70%
- 减少 CPU 占用

**使用场景**:
- Colo 机房高频交易
- 需要极低延迟的场景

**安装**:
```bash
cd XAPI/onload-8.0.0.34/
./scripts/onload_install
```

**使用**:
```bash
# 使用 Onload 运行程序
onload ./XTrader_0.9.3
```

### 2. TCPDirect

**用途**: 内核旁路库，比 Onload 更底层

**位置**: `XAPI/tcpdirect-8.0.1.2/`

**特点**:
- 零拷贝
- 更低延迟
- 需要专用网卡 (Solarflare)

## 在 CMake 中的使用

### 标准配置模板

```cmake
# YAML-CPP
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/YAML-CPP/0.8.0/include/)
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/YAML-CPP/0.8.0/lib)

# HP-Socket
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/HP-Socket/5.8.2/include/)
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/HP-Socket/5.8.2/lib)

# FMTLogger
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/FMTLogger/include/)

# SPDLog
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/SPDLog/include/)

# parallel_hashmap
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/parallel_hashmap/)

# CTP API (行情)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/CTP/v6.3.15/include/)
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/CTP/v6.3.15/lib/)

# 链接库
target_link_libraries(${APP_NAME}
    pthread
    hpsocket4c
    yaml-cpp
    dl
    thostmduserapi_se    # CTP 行情
    thosttraderapi_se    # CTP 交易
)
```

### 交易 API 动态加载

XTrader 使用插件架构，运行时动态加载柜台 API：

```cpp
// TraderAPI 插件编译为 .so
add_library(CTPTraderAPI SHARED CTPTradeGateWay.cpp)
target_link_libraries(CTPTraderAPI thosttraderapi_se)

// XTrader 运行时加载
void* handle = dlopen("libCTPTraderAPI.so", RTLD_LAZY);
```

## 版本兼容性

### CTP 版本选择

| 版本 | 适用场景 | 兼容性 |
|------|---------|--------|
| v6.3.15 | 生产环境稳定版 | 主流期货公司 |
| v6.6.1 | 较新功能 | 部分期货公司 |
| v6.7.2 | 最新功能 | 少数期货公司 |

**建议**: 根据期货公司要求选择对应版本

### ABI 兼容性要求

**所有柜台 API 使用旧 ABI**，QuantFabric 必须统一使用：
```cmake
add_definitions(-D_GLIBCXX_USE_CXX11_ABI=0)
```

## 许可证与使用限制

### 开源库
- YAML-CPP: MIT License
- SPDLog: MIT License
- HP-Socket: Apache 2.0
- parallel_hashmap: Apache 2.0

### 商业柜台 API
- CTP/REM/YD/OES/XTP/Tora: **需要向券商/期货公司申请**
- 仅限已签约客户使用
- 不可二次分发

### 网络加速库
- Solarflare Onload: 商业许可，需购买 Solarflare 网卡
- TCPDirect: 商业许可

## 常见问题

### 1. YAML-CPP 链接错误 (Ubuntu)
**问题**: `undefined reference to YAML::LoadFile`

**解决**:
```bash
cd XAPI/YAML-CPP/0.8.0/
mkdir build && cd build
cmake .. -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=0"
make
```

### 2. CTP API 版本不匹配
**问题**: 连接柜台失败，返回错误码 -1

**解决**: 咨询期货公司技术支持，使用正确的 API 版本

### 3. HP-Socket 编译错误
**问题**: 找不到 `libhpsocket4c.a`

**解决**: 确保使用正确的相对路径
```cmake
link_directories(${CMAKE_CURRENT_SOURCE_DIR}/../XAPI/HP-Socket/5.8.2/lib)
```

### 4. Onload 安装失败
**问题**: 内核模块加载失败

**解决**: 检查内核版本兼容性
```bash
uname -r  # 查看内核版本
# Onload 支持 Linux 3.x - 5.x
```

## 维护与更新

### 更新 XAPI 子模块
```bash
cd XAPI
git pull origin master
```

### 添加新的柜台 API
1. 在 `XAPI/` 下创建对应目录
2. 放置头文件和库文件
3. 更新各组件的 CMakeLists.txt
4. 实现对应的 GateWay 插件

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XAPI
