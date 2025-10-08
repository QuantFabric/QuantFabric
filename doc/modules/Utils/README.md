# Utils 文档索引

## 📚 文档列表

### 1. [模块概览](00_Overview.md)
快速了解 Utils 工具库

**内容**:
- 模块概述
- 核心组件（配置管理、网络通信、IPC、日志、消息协议、工具函数）
- 使用示例
- 编译配置

**适合**:
- 系统开发者
- 快速参考

## 🎯 核心组件

### 配置管理
- **YAMLConfig**: YAML 配置文件加载与解析
- **支持类型**: string, int, double, bool, array, map

### 网络通信
- **HPPackClient**: TCP Pack 客户端
- **HPPackServer**: TCP Pack 服务端
- **协议**: 4-byte 长度头 + 数据体

### 进程间通信
- **SHMServer**: 共享内存服务端
- **SHMConnection**: 共享内存客户端
- **特点**: 零拷贝、SPSC 无锁队列

### 日志系统
- **Logger**: 基于 SPDLog 的日志封装
- **级别**: Trace, Debug, Info, Warn, Error, Critical
- **输出**: 控制台 + 文件

### 消息协议
- **PackMessage**: 统一消息结构
- **类型**: 行情、订单、风控、系统等
- **序列化**: POD 结构，直接内存拷贝

### 工具函数
- **时间**: getCurrentTimeUs(), TimeUtils
- **字符串**: StringUtils, Trim, Split, Join
- **CPU 绑定**: ThreadBind(), SetAffinity()
- **文件**: FileExists(), CreateDirectory()

## 💡 使用示例

### YAML 配置加载
```cpp
Utils::YAMLConfig config;
config.LoadConfig("config.yml");

std::string ip = config["Server"]["IP"].as<std::string>();
int port = config["Server"]["Port"].as<int>();
```

### 日志使用
```cpp
Utils::Logger logger("MyApp");
logger.info("Application started");
logger.error("Connection failed: {}", errorMsg);
```

### CPU 绑定
```cpp
int cpuset = 10;
Utils::ThreadBind(pthread_self(), cpuset);
```

### 时间函数
```cpp
unsigned long now = Utils::getCurrentTimeUs();  // 微秒时间戳
std::string timeStr = Utils::getTimeStamp();    // 格式化时间
```

## 📖 相关文档

- [系统总体架构](../../architecture/SystemArchitecture.md) - Utils 在系统中的应用
- [部署运维指南](../../deployment/DeploymentGuide.md) - 工具使用

---
作者: @scorpiostudio @yutiansut
