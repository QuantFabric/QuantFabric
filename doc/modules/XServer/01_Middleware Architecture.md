# XServer 中间件架构详解

## 设计理念

### 核心定位

XServer 是 QuantFabric 的**消息路由中间件**，连接用户侧和 Colo 交易服务器。

**设计目标**:
1. **解耦**: 隔离用户网络与交易网络
2. **路由**: 根据 Colo/Account 路由消息
3. **权限**: 用户认证与权限管理
4. **回放**: 历史数据回放功能

### 架构图

```
用户侧                    XServer                     Colo 侧
┌────────┐              ┌────────────┐              ┌─────────┐
│XMonitor│◄───TCP─────►│ 路由引擎    │◄───TCP─────►│XWatcher │
│ (GUI)  │              │            │              │         │
└────────┘              │ ┌────────┐ │              └────┬────┘
                        │ │权限管理│ │                   │
┌────────┐              │ └────────┘ │              ┌────▼────┐
│XMonitor│              │            │              │XMarket  │
│ (GUI)  │              │ ┌────────┐ │              │Center   │
└────────┘              │ │消息队列│ │              └─────────┘
                        │ └────────┘ │              ┌─────────┐
                        │            │              │XTrader  │
                        │ ┌────────┐ │              └─────────┘
                        │ │数据回放│ │
                        │ └────────┘ │
                        └────────────┘
```

## 核心组件

### 1. 路由引擎

**消息路由表**:

```cpp
class XServer {
private:
    // Colo → XWatcher 连接映射
    std::unordered_map<std::string, CONNID> m_ColoConnMap;

    // Client UUID → Connection 映射
    std::unordered_map<std::string, CONNID> m_ClientConnMap;

    // Account → Colo 映射
    std::unordered_map<std::string, std::string> m_AccountColoMap;

public:
    // 路由消息
    void RouteMessage(Message::PackMessage& msg, const std::string& from) {
        switch (msg.MessageType) {
            case Message::EOrderRequest:
                // 报单请求：Client → Colo
                RouteToTrader(msg);
                break;

            case Message::EOrderStatus:
                // 订单回报：Colo → Client
                RouteToClient(msg);
                break;

            // ... 其他消息类型
        }
    }

private:
    void RouteToTrader(Message::PackMessage& msg) {
        // 1. 根据 Account 查找 Colo
        std::string colo = m_AccountColoMap[msg.OrderRequest.Account];

        // 2. 查找 XWatcher 连接
        CONNID connID = m_ColoConnMap[colo];

        // 3. 转发消息
        m_PackServer->SendData(connID, (unsigned char*)&msg, sizeof(msg));
    }

    void RouteToClient(Message::PackMessage& msg) {
        // 1. 查找订阅该消息的客户端
        for (auto& [uuid, connID] : m_ClientConnMap) {
            if (HasPermission(uuid, msg.MessageType)) {
                m_PackServer->SendData(connID, (unsigned char*)&msg, sizeof(msg));
            }
        }
    }
};
```

### 2. 权限管理

**数据库表结构** (SQLite):

```sql
CREATE TABLE users (
    account TEXT PRIMARY KEY,
    password TEXT NOT NULL,
    role TEXT NOT NULL,                 -- admin / trader / viewer
    plugins TEXT,                       -- Market|OrderManager|RiskJudge
    messages TEXT,                      -- FutureMarket|OrderStatus|AccountPosition
    colo TEXT,                          -- ColoSH / ColoBJ
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE permissions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    role TEXT NOT NULL,
    plugin TEXT NOT NULL,
    message TEXT NOT NULL,
    allow INTEGER DEFAULT 1             -- 1=allow, 0=deny
);
```

**权限检查流程**:

```cpp
class PermissionManager {
public:
    bool CheckLogin(const std::string& account, const std::string& password) {
        std::string sql = "SELECT password FROM users WHERE account = ?";
        SQLite::Statement query(m_DB, sql);
        query.bind(1, account);

        if (query.executeStep()) {
            std::string storedPwd = query.getColumn(0).getString();
            return VerifyPassword(password, storedPwd);  // bcrypt/sha256
        }
        return false;
    }

    bool HasPermission(const std::string& account,
                       const std::string& plugin,
                       const std::string& message) {
        // 1. 查询用户角色
        std::string role = GetUserRole(account);

        // 2. 查询角色权限
        std::string sql = R"(
            SELECT allow FROM permissions
            WHERE role = ? AND plugin = ? AND message = ?
        )";

        SQLite::Statement query(m_DB, sql);
        query.bind(1, role);
        query.bind(2, plugin);
        query.bind(3, message);

        if (query.executeStep()) {
            return query.getColumn(0).getInt() == 1;
        }

        return false;  // 默认拒绝
    }

private:
    SQLite::Database m_DB;
};
```

**登录流程**:

```
1. XMonitor 发送登录请求
   TLoginRequest {
       Account, Password, Plugins, Messages
   }

2. XServer 验证
   - 检查 Account/Password
   - 查询用户权限

3. XServer 响应
   TLoginResponse {
       ErrorID: 0=成功, 非0=失败
       Role, Plugins, Messages  // 实际权限
   }

4. XMonitor 根据权限显示插件
```

### 3. 消息队列

**异步处理机制**:

```cpp
class XServer {
private:
    // 消息队列（LockFreeQueue）
    LockFreeQueue<Message::PackMessage> m_MessageQueue;

    // 工作线程
    std::thread* m_WorkThread;

public:
    void Start() {
        // 启动工作线程
        m_WorkThread = new std::thread(&XServer::WorkFunc, this);
    }

private:
    // 接收回调（HPSocket 线程）
    EnHandleResult OnReceive(HP_TcpPackServer pSender, CONNID dwConnID,
                             const BYTE* pData, int iLength) {
        Message::PackMessage* msg = (Message::PackMessage*)pData;

        // 1. 写入队列（无锁）
        m_MessageQueue.Push(*msg);

        // 2. 立即返回（不阻塞 HPSocket 线程）
        return HR_OK;
    }

    // 工作线程（消息处理）
    void WorkFunc() {
        while (m_Running) {
            Message::PackMessage msg;
            if (m_MessageQueue.Pop(msg)) {
                // 路由消息
                RouteMessage(msg);

                // 持久化（可选）
                PersistMessage(msg);
            }
        }
    }
};
```

**优势**:
- HPSocket 线程快速返回，避免阻塞
- 工作线程处理业务逻辑（路由、权限、持久化）
- 解耦网络 I/O 与业务处理

### 4. 数据回放

**回放架构**:

```
历史数据存储:
┌─────────────────────────────────────┐
│  SQLite / ClickHouse / 文件系统      │
│  - market_data_{date}.db             │
│  - order_status_{date}.db            │
│  - account_position_{date}.db        │
└─────────────────────────────────────┘
          ↓ 回放引擎读取
┌─────────────────────────────────────┐
│      ReplayEngine                    │
│  - 按时间戳排序                       │
│  - 模拟发送                           │
│  - 速度控制 (1x / 10x / 实时)         │
└─────────────────────────────────────┘
          ↓ 通过 XServer 转发
┌─────────────────────────────────────┐
│      XMonitor (回放模式)              │
│  - 历史行情展示                       │
│  - 历史订单分析                       │
│  - 策略回测                           │
└─────────────────────────────────────┘
```

**回放实现**:

```cpp
class ReplayEngine {
public:
    void Replay(const std::string& date, double speed = 1.0) {
        // 1. 加载历史数据
        std::vector<Message::PackMessage> history;
        LoadHistory(date, history);

        // 2. 按时间戳排序
        std::sort(history.begin(), history.end(),
                  [](const auto& a, const auto& b) {
                      return a.Timestamp < b.Timestamp;
                  });

        // 3. 回放
        unsigned long startTime = Utils::getTimeUs();
        for (auto& msg : history) {
            // 计算应该发送的时间
            unsigned long sendTime = startTime +
                (msg.Timestamp - history[0].Timestamp) / speed;

            // 等待到发送时间
            unsigned long now = Utils::getTimeUs();
            if (now < sendTime) {
                usleep(sendTime - now);
            }

            // 发送消息
            m_Server->BroadcastMessage(msg);
        }
    }

private:
    void LoadHistory(const std::string& date,
                     std::vector<Message::PackMessage>& out) {
        std::string dbPath = fmt::format("./history/market_{}.db", date);
        SQLite::Database db(dbPath, SQLite::OPEN_READONLY);

        SQLite::Statement query(db, "SELECT * FROM market_data ORDER BY timestamp");

        while (query.executeStep()) {
            Message::PackMessage msg;
            msg.MessageType = Message::EFutureMarket;
            msg.Timestamp = query.getColumn("timestamp").getInt64();
            // ... 其他字段
            out.push_back(msg);
        }
    }
};
```

## 性能优化

### 1. 连接池

```cpp
class ConnectionPool {
public:
    CONNID AcquireConnection(const std::string& colo) {
        std::lock_guard<std::mutex> lock(m_Mutex);

        auto it = m_Pool.find(colo);
        if (it != m_Pool.end() && !it->second.empty()) {
            CONNID connID = it->second.back();
            it->second.pop_back();
            return connID;
        }

        // 创建新连接
        return CreateConnection(colo);
    }

    void ReleaseConnection(const std::string& colo, CONNID connID) {
        std::lock_guard<std::mutex> lock(m_Mutex);
        m_Pool[colo].push_back(connID);
    }

private:
    std::unordered_map<std::string, std::vector<CONNID>> m_Pool;
    std::mutex m_Mutex;
};
```

### 2. 批量处理

```cpp
void XServer::WorkFunc() {
    const int BATCH_SIZE = 100;
    Message::PackMessage batch[BATCH_SIZE];

    while (m_Running) {
        int count = 0;

        // 批量读取
        while (count < BATCH_SIZE && m_MessageQueue.Pop(batch[count])) {
            count++;
        }

        // 批量路由
        if (count > 0) {
            RouteBatch(batch, count);
        }
    }
}
```

## 总结

XServer 中间件通过以下机制实现高效消息路由：

1. **路由引擎**: 基于 Colo/Account/权限的消息路由
2. **权限管理**: SQLite 存储 + 角色权限模型
3. **异步队列**: 解耦网络 I/O 与业务处理
4. **数据回放**: 历史数据加载 + 时间模拟

**关键优势**:
- 解耦用户网络与交易网络
- 统一权限管理
- 支持多 Colo、多用户
- 历史回放能力

**下一步**: 阅读 XMonitor GUI 架构文档了解客户端实现。
