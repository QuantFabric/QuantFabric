# XServer 模块文档

## 模块概述

XServer 是 QuantFabric 的**中间件**，部署在用户侧或公司侧，负责转发 GUI 控制命令到 Colo 服务器，转发交易数据到 GUI 客户端，管理用户权限。

**项目地址**: [XServer](https://github.com/QuantFabric/XServer)

## 核心功能

1. **命令转发**: GUI → Colo 服务器
2. **数据转发**: Colo 服务器 → GUI
3. **用户权限管理**: 登录认证、权限校验
4. **历史数据回放**: 盘后回放交易数据

## 架构设计

```
XMonitor (GUI)
    ↓ 网络
XServer (中间件)
    ↓ 网络
XWatcher (Colo 服务器)
    ↓
交易组件
```

## 核心组件

### 1. 路由转发

**命令路由**:
```cpp
void RouteCommand(Message::PackMessage& msg, const std::string& colo) {
    // 转发到对应 Colo 的 XWatcher
    HPPackClient* watcherClient = GetWatcherClient(colo);
    watcherClient->SendData((unsigned char*)&msg, sizeof(msg));
}
```

**数据路由**:
```cpp
void RouteData(Message::PackMessage& msg, const std::string& clientUUID) {
    // 转发到对应 GUI 客户端
    HPPackServer::SendData(clientUUID, (unsigned char*)&msg, sizeof(msg));
}
```

### 2. 用户权限管理

**数据库表** (SQLite):
```sql
CREATE TABLE users (
    Account TEXT PRIMARY KEY,
    Password TEXT,
    Role TEXT,
    Plugins TEXT,      -- 可访问插件列表
    Messages TEXT,     -- 可订阅消息列表
    UpdateTime TEXT
);
```

**权限校验**:
```cpp
bool CheckPermission(const std::string& account, const std::string& plugin) {
    std::string plugins = GetUserPlugins(account);
    return plugins.find(plugin) != std::string::npos;
}
```

### 3. 历史回放

**功能**:
- 回放历史行情
- 回放历史订单
- 模拟交易环境

## 配置文件

```yaml
XServerConfig:
  ServerIP: 0.0.0.0
  ServerPort: 9000
  DatabasePath: ./XServer.db
```

## 运行方式

```bash
./XServer_0.9.3 XServer.yml
```

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XServer
