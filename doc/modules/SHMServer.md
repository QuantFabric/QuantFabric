# SHMServer 模块文档

## 模块概述

SHMServer 是基于共享内存的**进程间通信服务**，提供发布-订阅模式和 CS 模式通信，支持 C++ 和 Python (通过 pybind11 绑定)。

**项目地址**: [SHMServer](https://github.com/QuantFabric/SHMServer)

## 核心功能

1. **共享内存 IPC**: 低延迟进程间通信
2. **SPSC 队列**: 单生产者单消费者模式
3. **发布-订阅**: 一对多消息分发
4. **Python 绑定**: 支持 Python 策略开发

## 架构设计

### 通信模式

```
发布-订阅模式:
Publisher → SHMServer → Subscriber1
                     → Subscriber2
                     → Subscriber3

CS 模式:
Client1 → SHMServer → Client2
       ← SHMServer ←
```

### 通信信道

每个客户端分配独立信道:
- **发送队列** (SPSC): Server → Client
- **接收队列** (SPSC): Client → Server

## 核心组件

### 1. SHMServer (服务端)

**功能**:
- 创建共享内存
- 管理客户端连接
- 消息路由

**使用** (C++):
```cpp
#include "SHMServer.hpp"

class OrderServer : public SHMIPC::SHMServer<Message::PackMessage, ServerConf> {
    void OnMessage(Message::PackMessage& msg) override {
        // 处理消息
    }
};

OrderServer server;
server.Start("OrderServer", cpuset);
```

### 2. SHMConnection (客户端)

**功能**:
- 连接共享内存
- 发送/接收消息

**使用** (C++):
```cpp
#include "SHMConnection.hpp"

SHMIPC::SHMConnection<Message::PackMessage, ClientConf> conn("ClientID");
conn.Start("OrderServer");

// 发送
Message::PackMessage msg;
conn.Send(&msg, sizeof(msg));

// 接收
if (conn.Receive(&msg, sizeof(msg))) {
    // 处理消息
}
```

### 3. SPSCQueue (无锁队列)

**特点**:
- 单生产者单消费者
- 无锁实现
- 基于环形缓冲区

## Python 绑定

### 安装依赖

```bash
# 安装 pybind11
git clone https://github.com/pybind/pybind11.git
cd pybind11 && mkdir build && cd build
cmake .. && make && sudo make install

# 安装 Python 开发包
yum install python3-devel
```

### 编译 Python 扩展

```bash
cd SHMServer/pybind11
mkdir build && cd build
cmake .. && make
```

**输出**:
- `pack_message.cpython-39-x86_64-linux-gnu.so`
- `shm_server.cpython-39-x86_64-linux-gnu.so`
- `shm_connection.cpython-39-x86_64-linux-gnu.so`
- `spscqueue.cpython-39-x86_64-linux-gnu.so`

### Python 使用示例

```python
import sys
sys.path.append('/path/to/SHMServer/pybind11/build')

import shm_connection
import pack_message

# 连接
conn = shm_connection.SHMConnection("MyClient")
conn.start("MarketServer")

# 接收行情
while True:
    data = conn.receive()
    if data:
        print(f"LastPrice: {data['LastPrice']}")
```

## 性能特点

1. **零拷贝**: 共享内存直接访问
2. **无锁队列**: 避免互斥锁开销
3. **低延迟**: 微秒级通信

**性能指标**:
- C++ 通信延迟: 2-5μs
- Python 通信延迟: 50-200μs

## 测试用例

**C++ 测试**:
```bash
cd SHMServer
g++ -std=c++17 CPP_IPC_Test.cpp -o test -lpthread
./test
```

**Python 测试**:
```bash
cd SHMServer/pybind11
python3 shm_connection_test.py
python3 market_client_test.py
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/SHMServer
