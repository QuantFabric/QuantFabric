# XMonitor 模块文档

## 模块概述

XMonitor 是 QuantFabric 的 **Qt GUI 监控客户端**，提供可视化交易监控、手动报单/撤单、风控管理、用户权限管理等功能。

**项目地址**: [XMonitor](https://github.com/QuantFabric/XMonitor)

## 核心功能

1. **行情监控**: 实时行情展示
2. **订单管理**: 手动报单/撤单、订单查询
3. **仓位资金**: 实时仓位和资金监控
4. **风控管理**: 风控参数设置、撤单统计
5. **性能监控**: Colo 服务器性能和进程状态
6. **用户权限**: 用户和权限管理

## 插件架构

XMonitor 采用**可拖拽式插件架构**，基于 FinTechUI 框架。

### 插件列表

| 插件 | 功能 |
|------|-----|
| Permission | 用户权限管理、消息订阅 |
| Market | 行情数据展示 |
| EventLog | 事件日志展示 |
| Monitor | 服务器性能、进程管理 |
| RiskJudge | 风控参数设置、撤单统计 |
| OrderManager | 报单/撤单、订单/仓位/资金查询 |
| FutureAnalysis | 期货仓位分析 |
| StockAnalysis | 股票仓位分析 |

## 核心组件

### 1. HPPackClient (网络客户端)

**功能**:
- 连接 XServer
- 发送控制命令
- 接收交易数据

### 2. 插件管理

**插件接口**:
```cpp
class IPlugin {
public:
    virtual void OnMessage(const Message::PackMessage& msg) = 0;
    virtual QWidget* GetWidget() = 0;
};
```

**插件注册**:
```cpp
PluginManager::RegisterPlugin("Market", new MarketPlugin());
```

### 3. 数据模型

**XTableModel**:
- 自定义 Qt 表格模型
- 支持动态更新
- 支持排序和过滤

## 主要界面

### 1. Permission 插件

**功能**:
- 用户登录
- 插件权限配置
- 消息订阅配置

**界面元素**:
- 账户列表
- 插件权限复选框
- 消息类型复选框

### 2. Market 插件

**功能**:
- 实时行情展示
- 多 Ticker 监控

**显示字段**:
- Ticker、最新价、涨跌幅
- 买卖五档、成交量

### 3. OrderManager 插件

**功能**:
- 手动报单/撤单
- 订单查询
- 仓位查询
- 资金查询

**报单界面**:
- Ticker 输入
- 价格、数量输入
- 买卖方向选择
- 开平仓选择

### 4. RiskJudge 插件

**功能**:
- 流速控制参数设置
- 撤单限制参数设置
- 账户锁定/解锁
- 撤单次数统计

**界面元素**:
- 流速限制输入框
- Ticker 撤单限制输入框
- 账户锁定按钮
- 撤单次数表格

### 5. Monitor 插件

**功能**:
- Colo 服务器性能监控
- 交易进程状态监控
- 进程启动/停止

**监控指标**:
- CPU 使用率
- 内存使用率
- 进程运行状态

## 编译构建

### 依赖

- Qt 5.12+
- FinTechUI 组件库

### 编译

```bash
cd XMonitor
git submodule update --init --recursive
mkdir build && cd build
qmake ..
make -j8
```

**输出**: `build/XMonitor`

## 配置文件

### XMonitor.yml

```yaml
XMonitorConfig:
  ServerIP: 192.168.1.100       # XServer IP
  ServerPort: 9000
  Account: trader001
  Password: ******
  AutoLogin: true
```

## 运行方式

```bash
./XMonitor
```

## 界面截图参考

- Permission: 用户权限管理界面
- Market: 行情监控界面
- EventLog: 事件日志界面
- Monitor: 性能监控界面
- RiskJudge: 风控管理界面
- OrderManager: 订单管理界面

## 作者与维护

**创建者**: @scorpiostudio @yutiansut
**仓库**: https://github.com/QuantFabric/XMonitor
