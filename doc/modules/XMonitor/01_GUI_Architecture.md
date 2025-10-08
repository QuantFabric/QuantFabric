# XMonitor GUI 架构与插件系统

## 设计理念

### 核心目标

XMonitor 是 QuantFabric 的**可视化监控客户端**，采用 Qt 框架 + 插件架构。

**设计原则**:
1. **模块化**: 插件式架构，功能解耦
2. **可定制**: 用户自定义布局和插件
3. **高性能**: 实时更新海量行情数据
4. **易扩展**: 新增插件无需修改主框架

### 插件架构图

```
┌──────────────────────────────────────────────┐
│              XMonitor 主框架                  │
│  ┌────────────────────────────────────────┐  │
│  │   MainWindow (主窗口)                   │  │
│  │   ├─ MenuBar (菜单栏)                   │  │
│  │   ├─ ToolBar (工具栏)                   │  │
│  │   └─ CentralWidget (中心区域)           │  │
│  │       └─ PluginContainer (插件容器)     │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │   PluginManager (插件管理器)            │  │
│  │   ├─ LoadPlugin()                       │  │
│  │   ├─ UnloadPlugin()                     │  │
│  │   └─ RouteMessage()                     │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │   NetworkClient (网络客户端)            │  │
│  │   ├─ Connect to XServer                 │  │
│  │   ├─ Send/Receive Messages              │  │
│  │   └─ Heartbeat                          │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
                    ↓ 加载
┌──────────────────────────────────────────────┐
│                插件层                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐         │
│  │Permission│ │  Market │ │EventLog │         │
│  └─────────┘ └─────────┘ └─────────┘         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐         │
│  │ Monitor │ │RiskJudge│ │OrderMgr │         │
│  └─────────┘ └─────────┘ └─────────┘         │
└──────────────────────────────────────────────┘
```

## 核心组件

### 1. 插件接口定义

**IPlugin.h** - 插件基类:

```cpp
class IPlugin : public QObject {
    Q_OBJECT

public:
    virtual ~IPlugin() = default;

    // ========== 必须实现的接口 ==========
    virtual QString GetName() const = 0;         // 插件名称
    virtual QString GetVersion() const = 0;      // 插件版本
    virtual QWidget* GetWidget() = 0;            // 插件界面
    virtual void OnMessage(const Message::PackMessage& msg) = 0;  // 消息处理

    // ========== 可选实现的接口 ==========
    virtual void OnActivate() {}                 // 插件激活
    virtual void OnDeactivate() {}               // 插件停用
    virtual void OnSave() {}                     // 保存状态
    virtual void OnLoad() {}                     // 加载状态

signals:
    void SendMessage(const Message::PackMessage& msg);  // 发送消息

public slots:
    void ReceiveMessage(const Message::PackMessage& msg) {
        OnMessage(msg);
    }
};
```

### 2. 插件管理器

**PluginManager.cpp**:

```cpp
class PluginManager : public QObject {
    Q_OBJECT

public:
    void LoadPlugin(const QString& name, IPlugin* plugin) {
        m_Plugins[name] = plugin;

        // 连接信号槽
        connect(plugin, &IPlugin::SendMessage,
                this, &PluginManager::OnPluginSendMessage);

        connect(this, &PluginManager::BroadcastMessage,
                plugin, &IPlugin::ReceiveMessage);

        // 添加到 UI
        AddPluginToUI(plugin);
    }

    void UnloadPlugin(const QString& name) {
        auto it = m_Plugins.find(name);
        if (it != m_Plugins.end()) {
            it.value()->OnSave();
            delete it.value();
            m_Plugins.erase(it);
        }
    }

    void RouteMessage(const Message::PackMessage& msg) {
        // 根据消息类型路由到对应插件
        switch (msg.MessageType) {
            case Message::EFutureMarket:
                RouteToPlugin("Market", msg);
                break;
            case Message::EOrderStatus:
                RouteToPlugin("OrderManager", msg);
                break;
            // ... 其他消息类型
            default:
                // 广播给所有插件
                emit BroadcastMessage(msg);
        }
    }

signals:
    void BroadcastMessage(const Message::PackMessage& msg);

private slots:
    void OnPluginSendMessage(const Message::PackMessage& msg) {
        // 转发到网络客户端
        emit SendToServer(msg);
    }

private:
    void RouteToPlugin(const QString& name, const Message::PackMessage& msg) {
        auto it = m_Plugins.find(name);
        if (it != m_Plugins.end()) {
            it.value()->OnMessage(msg);
        }
    }

    QHash<QString, IPlugin*> m_Plugins;
};
```

### 3. 可拖拽布局

**PluginContainer (基于 QDockWidget)**:

```cpp
class PluginContainer : public QMainWindow {
    Q_OBJECT

public:
    void AddPlugin(IPlugin* plugin) {
        // 创建 DockWidget
        QDockWidget* dock = new QDockWidget(plugin->GetName(), this);
        dock->setWidget(plugin->GetWidget());

        // 添加到主窗口
        addDockWidget(Qt::TopDockWidgetArea, dock);

        // 保存引用
        m_DockWidgets[plugin->GetName()] = dock;
    }

    void SaveLayout() {
        QSettings settings("QuantFabric", "XMonitor");
        settings.setValue("geometry", saveGeometry());
        settings.setValue("windowState", saveState());
    }

    void LoadLayout() {
        QSettings settings("QuantFabric", "XMonitor");
        restoreGeometry(settings.value("geometry").toByteArray());
        restoreState(settings.value("windowState").toByteArray());
    }

private:
    QHash<QString, QDockWidget*> m_DockWidgets;
};
```

## 插件实现示例

### Market 插件

**MarketPlugin.cpp**:

```cpp
class MarketPlugin : public IPlugin {
    Q_OBJECT

public:
    MarketPlugin() {
        // 创建 UI
        m_Widget = new QWidget();
        m_TableView = new QTableView(m_Widget);
        m_Model = new MarketDataModel(m_Widget);

        m_TableView->setModel(m_Model);

        // 布局
        QVBoxLayout* layout = new QVBoxLayout(m_Widget);
        layout->addWidget(m_TableView);
    }

    QString GetName() const override { return "Market"; }
    QString GetVersion() const override { return "1.0.0"; }
    QWidget* GetWidget() override { return m_Widget; }

    void OnMessage(const Message::PackMessage& msg) override {
        if (msg.MessageType == Message::EFutureMarket) {
            // 更新模型
            m_Model->UpdateMarketData(msg.FutureMarketData);

            // 刷新视图
            m_TableView->viewport()->update();
        }
    }

private:
    QWidget* m_Widget;
    QTableView* m_TableView;
    MarketDataModel* m_Model;
};
```

**MarketDataModel (自定义表格模型)**:

```cpp
class MarketDataModel : public QAbstractTableModel {
    Q_OBJECT

public:
    int rowCount(const QModelIndex&) const override {
        return m_Data.size();
    }

    int columnCount(const QModelIndex&) const override {
        return 10;  // Ticker, LastPrice, Volume, ...
    }

    QVariant data(const QModelIndex& index, int role) const override {
        if (role == Qt::DisplayRole) {
            const auto& item = m_Data[index.row()];
            switch (index.column()) {
                case 0: return QString::fromLocal8Bit(item.Ticker);
                case 1: return item.LastPrice;
                case 2: return item.Volume;
                // ... 其他列
            }
        }
        return QVariant();
    }

    void UpdateMarketData(const MarketData::TFutureMarketData& data) {
        // 查找或插入
        auto it = m_TickerIndexMap.find(data.Ticker);
        if (it == m_TickerIndexMap.end()) {
            int row = m_Data.size();
            beginInsertRows(QModelIndex(), row, row);
            m_Data.push_back(data);
            m_TickerIndexMap[data.Ticker] = row;
            endInsertRows();
        } else {
            int row = it->second;
            m_Data[row] = data;
            emit dataChanged(index(row, 0), index(row, columnCount() - 1));
        }
    }

private:
    std::vector<MarketData::TFutureMarketData> m_Data;
    std::unordered_map<std::string, int> m_TickerIndexMap;
};
```

### OrderManager 插件

**OrderManagerPlugin.cpp**:

```cpp
class OrderManagerPlugin : public IPlugin {
public:
    OrderManagerPlugin() {
        // 创建 Tab Widget
        m_TabWidget = new QTabWidget();

        // 报单面板
        m_OrderPanel = new OrderPanel();
        m_TabWidget->addTab(m_OrderPanel, "报单");

        // 订单列表
        m_OrderList = new OrderListWidget();
        m_TabWidget->addTab(m_OrderList, "订单");

        // 仓位列表
        m_PositionList = new PositionListWidget();
        m_TabWidget->addTab(m_PositionList, "仓位");

        // 连接信号
        connect(m_OrderPanel, &OrderPanel::SendOrder,
                this, &OrderManagerPlugin::OnSendOrder);
    }

    QWidget* GetWidget() override { return m_TabWidget; }

    void OnMessage(const Message::PackMessage& msg) override {
        switch (msg.MessageType) {
            case Message::EOrderStatus:
                m_OrderList->UpdateOrder(msg.OrderStatus);
                break;
            case Message::EAccountPosition:
                m_PositionList->UpdatePosition(msg.AccountPosition);
                break;
        }
    }

private slots:
    void OnSendOrder(const Message::TOrderRequest& req) {
        Message::PackMessage msg;
        msg.MessageType = Message::EOrderRequest;
        msg.OrderRequest = req;
        emit SendMessage(msg);
    }

private:
    QTabWidget* m_TabWidget;
    OrderPanel* m_OrderPanel;
    OrderListWidget* m_OrderList;
    PositionListWidget* m_PositionList;
};
```

## 性能优化

### 1. 虚拟列表 (Virtual List)

```cpp
// 只渲染可见行，避免渲染数万行数据
class VirtualTableView : public QTableView {
protected:
    void paintEvent(QPaintEvent* event) override {
        QPainter painter(viewport());

        // 计算可见范围
        int firstRow = verticalScrollBar()->value();
        int lastRow = firstRow + viewport()->height() / rowHeight();

        // 只绘制可见行
        for (int row = firstRow; row <= lastRow && row < model()->rowCount(); ++row) {
            DrawRow(painter, row);
        }
    }
};
```

### 2. 批量更新

```cpp
class MarketDataModel {
public:
    void BatchUpdate(const std::vector<MarketData::TFutureMarketData>& batch) {
        // 关闭信号
        blockSignals(true);

        for (const auto& data : batch) {
            UpdateMarketData(data);
        }

        // 恢复信号，触发一次刷新
        blockSignals(false);
        emit dataChanged(index(0, 0), index(rowCount() - 1, columnCount() - 1));
    }
};
```

### 3. 异步加载

```cpp
class HistoryLoader : public QThread {
    Q_OBJECT

protected:
    void run() override {
        // 后台加载历史数据
        std::vector<Message::PackMessage> history;
        LoadFromDatabase(history);

        // 批量发送到主线程
        emit HistoryLoaded(history);
    }

signals:
    void HistoryLoaded(const std::vector<Message::PackMessage>& history);
};
```

## 总结

XMonitor 通过以下机制实现高效 GUI：

1. **插件架构**: 模块化设计，易于扩展
2. **信号槽机制**: Qt 信号槽实现松耦合
3. **可拖拽布局**: QDockWidget 实现灵活布局
4. **虚拟渲染**: 只渲染可见部分，提升性能
5. **异步加载**: 后台线程加载数据，避免 UI 卡顿

**关键优势**:
- 用户自定义布局
- 实时更新海量数据
- 插件式扩展
- 跨平台 (Windows/Linux)

**下一步**: 查看具体插件实现文档了解各插件功能。
