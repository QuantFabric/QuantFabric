# SHMServer 架构与原理

## 设计理念

### 1. 为什么需要共享内存IPC？

#### 传统 IPC 方式的局限

| 方式 | 延迟 | 带宽 | 缺点 |
|------|------|------|------|
| Socket | 10-50μs | 中 | 内核拷贝、系统调用开销 |
| 管道 (Pipe) | 5-20μs | 低 | 单向通信、缓冲区限制 |
| 消息队列 (MQ) | 10-30μs | 中 | 内核拷贝、队列满阻塞 |
| **共享内存** | **1-5μs** | **高** | **需要同步机制** |

#### 量化交易的需求

```
行情接收路径：
交易所 → 柜台 API → XMarketCenter → 共享内存 → XQuant → 策略计算

关键指标：
- 总延迟目标: <100μs
- IPC 部分: <5μs（占比 5%）
- 数据速率: 每秒数千次行情更新
```

**结论**: Socket/MQ 的 10-50μs 延迟无法满足要求，必须使用共享内存。

### 2. 核心设计目标

#### 2.1 极致低延迟

**零拷贝设计**:
```cpp
// 传统 Socket 方式（2次拷贝）
用户空间A → 内核缓冲区 → 用户空间B

// 共享内存方式（0次拷贝）
用户空间A → 共享内存 ← 用户空间B
```

**无锁同步**:
```cpp
// 使用原子操作替代互斥锁
std::atomic<size_t> head;  // 写指针
std::atomic<size_t> tail;  // 读指针

// 无锁 Push
bool Push(const T& item) {
    size_t h = head.load(std::memory_order_relaxed);
    size_t next = (h + 1) % capacity;
    if (next == tail.load(std::memory_order_acquire))
        return false;
    buffer[h] = item;
    head.store(next, std::memory_order_release);
    return true;
}
```

#### 2.2 高可靠性

**进程隔离**:
- 一个进程崩溃不影响共享内存
- 其他进程可继续访问

**异常恢复**:
```cpp
// 检测共享内存是否可用
if (shmctl(shmid, IPC_STAT, &buf) == -1) {
    // 共享内存异常，重新创建
    Recreate();
}
```

#### 2.3 多语言支持

**C++ 原生实现**:
```cpp
SHMIPC::SHMConnection<MarketData, ClientConf> conn;
conn.Start("MarketServer");
MarketData data;
conn.Receive(&data, sizeof(data));
```

**Python 绑定（pybind11）**:
```python
import shm_connection
conn = shm_connection.SHMConnection("Client1")
conn.start("MarketServer")
data = conn.receive()
```

## 架构设计

### 1. 整体架构

```
┌─────────────────────────────────────────┐
│          应用层                          │
│  XMarketCenter / XTrader / XQuant        │
├─────────────────────────────────────────┤
│         SHMServer 层                     │
│  ┌─────────┐  ┌──────────┐              │
│  │SHMServer│  │SHMConnection│            │
│  └────┬────┘  └────┬─────┘              │
├───────┼────────────┼─────────────────────┤
│      SPSC Queue    │                     │
│  ┌────▼────────────▼────┐               │
│  │   共享内存段         │               │
│  │  (System V / POSIX)  │               │
│  └─────────────────────┘                │
├─────────────────────────────────────────┤
│            内核层                        │
│   /dev/shm (tmpfs) / /proc/sysvipc      │
└─────────────────────────────────────────┘
```

### 2. 通信模式

#### 2.1 发布-订阅 (Pub-Sub)

```
Publisher (XMarketCenter)
    ↓ SHMServer
    ├─> Channel 1 → Subscriber 1 (XQuant-1)
    ├─> Channel 2 → Subscriber 2 (XQuant-2)
    └─> Channel N → Subscriber N (XQuant-N)
```

**特点**:
- 一对多广播
- 订阅者独立通道
- 无消息丢失（队列足够大）

#### 2.2 请求-响应 (Req-Rep)

```
Client (XTrader)
    ↓ Request Queue
RiskServer (XRiskJudge)
    ↓ Response Queue
Client (XTrader)
```

**特点**:
- 双向通信
- 每个客户端独立双向队列
- 支持同步/异步请求

### 3. 内存布局

#### 3.1 共享内存段结构

```
┌─────────────────────────────────────────┐  ← SHM 起始地址
│  Control Block (控制块)                  │
│  ┌────────────────────────────────────┐ │
│  │ Version: 1.0                       │ │
│  │ ServerStatus: RUNNING              │ │
│  │ ChannelCount: 3                    │ │
│  │ MaxChannels: 16                    │ │
│  │ CreateTime: 1736909876             │ │
│  └────────────────────────────────────┘ │
├─────────────────────────────────────────┤
│  Channel 0 (Client: XQuant-1)           │
│  ┌──────────────────────────────────┐  │
│  │  Send Queue (Server → Client)     │  │
│  │    Head: 123                      │  │
│  │    Tail: 100                      │  │
│  │    Capacity: 1024                 │  │
│  │    Buffer: [...市场数据...]        │  │
│  ├──────────────────────────────────┤  │
│  │  Recv Queue (Client → Server)     │  │
│  │    Head: 45                       │  │
│  │    Tail: 45                       │  │
│  │    Capacity: 256                  │  │
│  │    Buffer: [...控制命令...]        │  │
│  └──────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  Channel 1 (Client: XQuant-2)           │
│  ┌──────────────────────────────────┐  │
│  │  Send Queue                       │  │
│  │  ...                              │  │
│  ├──────────────────────────────────┤  │
│  │  Recv Queue                       │  │
│  │  ...                              │  │
│  └──────────────────────────────────┘  │
├─────────────────────────────────────────┤
│  ...                                    │
│  (更多 Channel)                          │
└─────────────────────────────────────────┘  ← SHM 结束地址
```

#### 3.2 SPSC 队列结构

```cpp
template<typename T>
struct SPSCQueue {
    alignas(64) std::atomic<size_t> head;    // 写指针（缓存行对齐）
    char padding1[64 - sizeof(std::atomic<size_t>)];

    alignas(64) std::atomic<size_t> tail;    // 读指针（缓存行对齐）
    char padding2[64 - sizeof(std::atomic<size_t>)];

    size_t capacity;
    T buffer[0];  // 柔性数组（实际大小由 capacity 决定）
};
```

**缓存行对齐原理**:
```
CPU 缓存行大小: 64 字节

未对齐:
[head][tail] ← 同一个缓存行
生产者写 head → 缓存行失效 → 消费者读 tail 缓存缺失 → 伪共享！

对齐后:
[head][padding...] ← 缓存行 1
[tail][padding...] ← 缓存行 2
生产者写 head → 缓存行 1 失效 → 不影响缓存行 2 → 无伪共享！
```

## 核心实现

### 1. SHMServer (服务端)

```cpp
template<typename T, typename Conf>
class SHMServer {
public:
    void Start(const std::string& name, int cpuset = -1) {
        // 1. 创建共享内存
        key_t key = ftok(name.c_str(), 0);
        m_SHMID = shmget(key, SHM_SIZE, IPC_CREAT | 0666);
        if (m_SHMID == -1) {
            throw std::runtime_error("shmget failed");
        }

        // 2. 映射到进程地址空间
        m_SHMAddr = (char*)shmat(m_SHMID, nullptr, 0);
        if (m_SHMAddr == (char*)-1) {
            throw std::runtime_error("shmat failed");
        }

        // 3. 初始化控制块
        ControlBlock* ctrl = (ControlBlock*)m_SHMAddr;
        ctrl->version = 1;
        ctrl->serverStatus = RUNNING;
        ctrl->channelCount = 0;
        ctrl->maxChannels = Conf::MaxChannels;
        ctrl->createTime = time(nullptr);

        // 4. 启动工作线程
        m_WorkThread = new std::thread(&SHMServer::WorkFunc, this);

        // 5. CPU 绑定
        if (cpuset >= 0) {
            Utils::ThreadBind(m_WorkThread->native_handle(), cpuset);
        }
    }

    void Publish(const T& data) {
        ControlBlock* ctrl = (ControlBlock*)m_SHMAddr;

        // 遍历所有订阅者
        for (int i = 0; i < ctrl->channelCount; i++) {
            Channel* channel = GetChannel(i);
            SPSCQueue<T>* queue = &channel->sendQueue;

            // 无锁写入
            if (!queue->Push(data)) {
                // 队列满，记录日志（不阻塞）
                m_Logger->Log->warn("Channel {} queue full", i);
            }
        }
    }

private:
    void WorkFunc() {
        while (m_Running) {
            // 处理客户端请求
            for (int i = 0; i < ctrl->channelCount; i++) {
                Channel* channel = GetChannel(i);
                SPSCQueue<T>* recvQueue = &channel->recvQueue;

                T request;
                if (recvQueue->Pop(request)) {
                    // 处理请求（虚函数，子类实现）
                    OnMessage(request);
                }
            }

            // 避免 CPU 空转
            std::this_thread::yield();
        }
    }

    virtual void OnMessage(const T& msg) = 0;  // 子类实现

    int m_SHMID;
    char* m_SHMAddr;
    std::thread* m_WorkThread;
    bool m_Running = true;
};
```

### 2. SHMConnection (客户端)

```cpp
template<typename T, typename Conf>
class SHMConnection {
public:
    void Start(const std::string& name) {
        // 1. 连接共享内存
        key_t key = ftok(name.c_str(), 0);
        m_SHMID = shmget(key, 0, 0);
        if (m_SHMID == -1) {
            throw std::runtime_error("shmget failed: server not started");
        }

        m_SHMAddr = (char*)shmat(m_SHMID, nullptr, 0);

        // 2. 分配 Channel
        m_ChannelID = AllocateChannel();

        // 3. 获取队列指针
        Channel* channel = GetChannel(m_ChannelID);
        m_SendQueue = &channel->recvQueue;  // Client→Server
        m_RecvQueue = &channel->sendQueue;  // Server→Client
    }

    bool Send(const T& data) {
        return m_SendQueue->Push(data);
    }

    bool Receive(T& data) {
        return m_RecvQueue->Pop(data);
    }

private:
    int AllocateChannel() {
        ControlBlock* ctrl = (ControlBlock*)m_SHMAddr;

        // 原子分配 Channel ID
        int channelID = ctrl->channelCount.fetch_add(1, std::memory_order_relaxed);

        if (channelID >= ctrl->maxChannels) {
            throw std::runtime_error("Too many clients");
        }

        // 初始化 Channel
        Channel* channel = GetChannel(channelID);
        channel->clientID = m_ClientID;
        channel->connectTime = time(nullptr);

        return channelID;
    }

    int m_SHMID;
    char* m_SHMAddr;
    int m_ChannelID;
    std::string m_ClientID;
    SPSCQueue<T>* m_SendQueue;
    SPSCQueue<T>* m_RecvQueue;
};
```

## 性能优化

### 1. 无锁算法

#### SPSC Queue 无锁实现

```cpp
template<typename T>
class SPSCQueue {
public:
    bool Push(const T& item) {
        // 生产者单线程，使用 relaxed
        size_t h = head.load(std::memory_order_relaxed);
        size_t next = (h + 1) % capacity;

        // 检查队列满（acquire 确保看到最新的 tail）
        if (next == tail.load(std::memory_order_acquire)) {
            return false;
        }

        // 写入数据
        buffer[h] = item;

        // 更新 head（release 确保写入对消费者可见）
        head.store(next, std::memory_order_release);

        return true;
    }

    bool Pop(T& item) {
        // 消费者单线程，使用 relaxed
        size_t t = tail.load(std::memory_order_relaxed);

        // 检查队列空（acquire 确保看到最新的 head）
        if (t == head.load(std::memory_order_acquire)) {
            return false;
        }

        // 读取数据
        item = buffer[t];

        // 更新 tail（release 确保读取完成）
        tail.store((t + 1) % capacity, std::memory_order_release);

        return true;
    }

private:
    alignas(64) std::atomic<size_t> head;
    alignas(64) std::atomic<size_t> tail;
    size_t capacity;
    T* buffer;
};
```

#### 内存序详解

| 内存序 | 作用 | 使用场景 |
|--------|------|----------|
| `relaxed` | 仅保证原子性 | 读写自己的指针 |
| `acquire` | 读操作后的读写不重排到前 | 读对方的指针 |
| `release` | 写操作前的读写不重排到后 | 写自己的指针给对方读 |
| `seq_cst` | 全局顺序一致 | 通用（性能最差） |

**SPSC 队列的内存序选择**:
```cpp
// 生产者
head.load(relaxed)     // 读自己的 head
tail.load(acquire)     // 读消费者的 tail（需要看到最新值）
head.store(release)    // 写 head 给消费者读（保证数据写入完成）

// 消费者
tail.load(relaxed)     // 读自己的 tail
head.load(acquire)     // 读生产者的 head（需要看到最新值）
tail.store(release)    // 写 tail 给生产者读（保证数据读取完成）
```

### 2. 缓存优化

#### 避免伪共享

```cpp
// 错误示例：伪共享
struct BadQueue {
    std::atomic<size_t> head;  // 假设位于缓存行 0x100
    std::atomic<size_t> tail;  // 假设位于缓存行 0x100（同一行！）
};
// 生产者写 head → 整个缓存行失效 → 消费者读 tail 缓存缺失

// 正确示例：缓存行对齐
struct GoodQueue {
    alignas(64) std::atomic<size_t> head;  // 缓存行 0x100
    char padding[56];                       // 填充到 64 字节
    alignas(64) std::atomic<size_t> tail;  // 缓存行 0x140
};
// 生产者写 head → 缓存行 0x100 失效 → 不影响缓存行 0x140
```

#### 数据预取

```cpp
// 批量处理时预取下一个数据
void ProcessBatch(SPSCQueue<T>& queue, int batchSize) {
    T items[batchSize];
    int count = 0;

    while (count < batchSize && queue.Pop(items[count])) {
        // 预取下一个（如果还有）
        if (count + 1 < batchSize) {
            __builtin_prefetch(&items[count + 1], 0, 3);
        }

        // 处理当前数据
        Process(items[count]);
        count++;
    }
}
```

### 3. 系统级优化

#### 大页内存 (Huge Pages)

```bash
# 配置大页
echo 1024 > /proc/sys/vm/nr_hugepages

# 使用大页共享内存
m_SHMID = shmget(key, SHM_SIZE, IPC_CREAT | SHM_HUGETLB | 0666);
```

**优势**:
- 减少 TLB (Translation Lookaside Buffer) 缺失
- 2MB 大页 vs 4KB 普通页 → TLB 缓存效率提升 500 倍

#### 内存锁定 (mlock)

```cpp
// 防止共享内存被 swap 到磁盘
void Start(const std::string& name) {
    m_SHMAddr = (char*)shmat(m_SHMID, nullptr, 0);

    // 锁定内存
    if (mlock(m_SHMAddr, SHM_SIZE) == -1) {
        perror("mlock failed");
    }
}
```

**优势**:
- 避免页错误 (Page Fault)
- 保证低延迟访问

## 总结

SHMServer 通过以下技术实现微秒级 IPC：

1. **共享内存**: 零拷贝数据传输
2. **无锁队列**: SPSC 模型 + 原子操作 + 内存序
3. **缓存优化**: 缓存行对齐 + 数据预取
4. **系统优化**: 大页内存 + 内存锁定

**性能对比**:
- Socket: 10-50μs
- SHMServer: 1-5μs
- **提升**: 10-50 倍

**下一步**: 阅读 [Python 绑定实现](02_PythonBinding.md) 了解如何在 Python 中使用。
