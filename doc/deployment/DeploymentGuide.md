# QuantFabric 部署运维指南

## 部署架构

### 典型部署方案

```
┌─────────────────────────────┐
│      用户办公网络            │
│  ┌────────┐   ┌──────────┐  │
│  │XMonitor│   │ XServer  │  │
│  └────────┘   └─────┬────┘  │
└────────────────────┬─────────┘
                     │ VPN/专线
┌────────────────────┼─────────┐
│     Colo 托管机房   │         │
│  ┌─────────────────┴──────┐  │
│  │      XWatcher          │  │
│  └─┬────────────────────┬─┘  │
│    │                    │    │
│  ┌─▼──────────┐  ┌──────▼─┐  │
│  │XMarket     │  │XTrader │  │
│  │Center      │  │        │  │
│  └────────────┘  └────┬───┘  │
│  ┌─────────────┐      │      │
│  │XRiskJudge   │◄─────┘      │
│  └─────────────┘              │
│  ┌─────────────┐              │
│  │   XQuant    │              │
│  └─────────────┘              │
└──────────────────────────────┘
```

## 环境准备

### 硬件要求

**Colo 交易服务器** (推荐配置):
- CPU: Intel Xeon E5/AMD EPYC, ≥8 核, 主频 ≥3.0GHz
- 内存: ≥32GB DDR4
- 硬盘: SSD ≥256GB
- 网络: 万兆网卡 (Solarflare 更佳)
- 操作系统: CentOS 7/8 或 Ubuntu 20.04/22.04

**用户侧服务器**:
- CPU: 4 核+
- 内存: 8GB+
- 硬盘: 100GB+
- 网络: 千兆

### 软件依赖

**系统软件** (CentOS):
```bash
# 编译工具链
yum install -y gcc gcc-c++ cmake make git

# 开发库
yum install -y sqlite-devel python3-devel

# 网络工具
yum install -y net-tools telnet wget curl
```

**系统软件** (Ubuntu):
```bash
# 编译工具链
apt-get install -y build-essential cmake git

# 开发库
apt-get install -y libsqlite3-dev python3-dev

# 网络工具
apt-get install -y net-tools telnet wget curl
```

## 编译部署

### 1. 下载源码

```bash
# 克隆主仓库及所有子模块
git clone --recurse-submodules git@github.com:QuantFabric/QuantFabric.git
cd QuantFabric

# 如果已克隆，更新子模块
git submodule init
git submodule update --remote
```

### 2. 编译 QuantFabric

```bash
# 执行编译脚本
sh build_release.sh

# 或手动编译
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j8
```

**编译输出** (build/ 目录):
- `XWatcher_0.9.3`
- `XMarketCenter_0.9.3`
- `XTrader_0.9.3`
- `XRiskJudge_0.9.3`
- `XQuant_0.9.3`
- `XServer_0.9.3`
- `libCTPMarketGateWay.so`
- `libCTPTradeGateWay.so`
- (其他插件 .so 文件)

### 3. 编译 XMonitor (Qt)

```bash
cd XMonitor
git pull
git submodule update --init --recursive

mkdir build && cd build
qmake ..
make -j8

# 输出: build/XMonitor
```

### 4. 编译 SHMServer Python 绑定 (可选)

```bash
# 安装 pybind11
cd /tmp
git clone https://github.com/pybind/pybind11.git
cd pybind11 && mkdir build && cd build
cmake .. && make && sudo make install

# 编译 Python 绑定
cd QuantFabric/SHMServer/pybind11
mkdir build && cd build
cmake .. && make

# 输出: *.so 文件 (Python 扩展模块)
```

## 配置文件

### 目录结构

```
QuantFabric/
├── build/              # 编译输出
├── config/             # 配置文件 (需创建)
│   ├── XWatcher.yml
│   ├── XMarketCenter.yml
│   ├── XTrader.yml
│   ├── XRiskJudge.yml
│   ├── XQuant.yml
│   ├── XServer.yml
│   ├── CTP.yml
│   └── TickerList.yml
├── logs/               # 日志目录 (自动创建)
└── scripts/            # 启动脚本 (需创建)
```

### 配置模板

#### XWatcher.yml

```yaml
XWatcherConfig:
  ServerIP: 192.168.1.100       # XServer IP
  ServerPort: 9000
  ColoName: ColoSH              # Colo 机房名称
  MonitorInterval: 5            # 性能监控间隔 (秒)
```

#### XMarketCenter.yml

```yaml
XMarketCenterConfig:
  ExchangeID: SHFE              # 交易所
  ServerIP: 127.0.0.1           # XWatcher IP
  Port: 8001
  TotalTick: 10000
  MarketServer: MarketServer    # 共享内存名称
  RecvTimeOut: 300
  APIConfig: ./config/CTP.yml   # 柜台配置
  TickerListPath: ./config/TickerList.yml
  ToMonitor: true
  Future: true
  CPUSET: 5                     # CPU 绑定
```

#### CTP.yml (行情)

```yaml
CTPMarketSource:
  FrontAddr: tcp://180.168.146.187:10131
  BrokerID: 9999
  Account: your_account
  Password: your_password
  TickerListPath: ./config/TickerList.yml
```

#### XTrader.yml

```yaml
XTraderConfig:
  Product: Futures
  BrokerID: 9999
  Account: your_account
  AppID: simnow_client_test
  AuthCode: 0000000000000000
  ServerIP: 127.0.0.1
  Port: 8001
  OpenTime: 08:30:00
  CloseTime: 15:30:00
  TickerListPath: ./config/TickerList.yml
  ErrorPath: ./config/CTPError.yml
  RiskServerName: RiskServer
  OrderServerName: OrderServer
  TraderAPI: ./config/CTP.yml
  CPUSET: [10, 11]
```

#### CTP.yml (交易)

```yaml
CTPTrader:
  FrontTradeAddr: tcp://180.168.146.187:10130
  BrokerID: 9999
  Account: your_account
  Password: your_password
  AppID: simnow_client_test
  AuthCode: 0000000000000000
```

#### XRiskJudge.yml

```yaml
XRiskJudgeConfig:
  Account: your_account
  ServerIP: 127.0.0.1
  Port: 8001
  RiskServerName: RiskServer
  CPUSET: 12
  RiskJudge:
    FlowLimit: 10
    TickerCancelLimit: 100
    OrderCancelLimit: 5
    LockAccount: false
    CheckSelfTrade: true
```

#### XQuant.yml

```yaml
XQuantConfig:
  XWatcherIP: 127.0.0.1
  XWatcherPort: 8001
  StrategyName: TestStrategy
  AccountList: your_account
  TickerListPath: ./config/TickerList.yml
  MarketServerName: MarketServer
  OrderServerName: OrderServer
  CPUSET: 17
  SnapshotInterval: 0
  SlicePerSec: 2
  KLineIntervals: 60, 300
  TradingSection:
    - 21:00:00-23:30:00
    - 09:00:00-10:15:00
    - 10:30:00-11:30:00
    - 13:30:00-15:00:00
```

#### TickerList.yml

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

#### XServer.yml

```yaml
XServerConfig:
  ServerIP: 0.0.0.0
  ServerPort: 9000
  DatabasePath: ./XServer.db
```

## 系统优化

### 1. 共享内存配置

```bash
# 查看当前配置
ipcs -l

# 临时修改
sysctl -w kernel.shmmax=17179869184    # 16GB
sysctl -w kernel.shmall=4194304        # 页数

# 永久修改 /etc/sysctl.conf
echo "kernel.shmmax=17179869184" >> /etc/sysctl.conf
echo "kernel.shmall=4194304" >> /etc/sysctl.conf
sysctl -p
```

### 2. CPU 隔离

```bash
# 编辑 /etc/default/grub
GRUB_CMDLINE_LINUX="isolcpus=5-20"

# 更新 grub
grub2-mkconfig -o /boot/grub2/grub.cfg

# 重启
reboot
```

**CPU 分配建议**:
- CPU 0-4: 系统保留
- CPU 5: XMarketCenter
- CPU 10: XTrader WorkThread
- CPU 11: XTrader OrderServer
- CPU 12: XRiskJudge
- CPU 17: XQuant

### 3. 网络优化

```bash
# /etc/sysctl.conf
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 87380 134217728
net.ipv4.tcp_wmem=4096 65536 134217728
net.core.netdev_max_backlog=5000

sysctl -p
```

### 4. 文件描述符

```bash
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535

# 重新登录生效
```

## 启动流程

### 1. Colo 服务器启动

**创建启动脚本** (`start_trading.sh`):
```bash
#!/bin/bash
cd /home/trader/QuantFabric/build

# 1. 启动 XWatcher
taskset -c 0 ./XWatcher_0.9.3 ../config/XWatcher.yml &
sleep 2

# 2. 启动 XMarketCenter
taskset -c 5 ./XMarketCenter_0.9.3 ../config/XMarketCenter.yml ./libCTPMarketGateWay.so &
sleep 2

# 3. 启动 XRiskJudge
taskset -c 12 ./XRiskJudge_0.9.3 ../config/XRiskJudge.yml &
sleep 2

# 4. 启动 XTrader
taskset -c 10 ./XTrader_0.9.3 ../config/XTrader.yml ./libCTPTradeGateWay.so &
sleep 2

# 5. 启动 XQuant (可选)
# taskset -c 17 ./XQuant_0.9.3 ../config/XQuant.yml &

echo "Trading system started"
```

**执行启动**:
```bash
chmod +x start_trading.sh
./start_trading.sh
```

### 2. 用户侧启动

```bash
# 启动 XServer
cd /path/to/XServer/build
./XServer_0.9.3 ../config/XServer.yml &

# 启动 XMonitor
cd /path/to/XMonitor/build
./XMonitor
```

## 进程管理

### systemd 服务配置

**XWatcher.service**:
```ini
[Unit]
Description=XWatcher Monitoring Service
After=network.target

[Service]
Type=simple
User=trader
WorkingDirectory=/home/trader/QuantFabric/build
ExecStart=/usr/bin/taskset -c 0 /home/trader/QuantFabric/build/XWatcher_0.9.3 ../config/XWatcher.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**启用服务**:
```bash
sudo systemctl daemon-reload
sudo systemctl enable XWatcher
sudo systemctl start XWatcher
sudo systemctl status XWatcher
```

## 监控与调试

### 1. 日志查看

```bash
# 实时查看日志
tail -f logs/XTrader_20250115.log

# 搜索错误
grep -i error logs/XTrader_20250115.log

# 查看最近 100 行
tail -100 logs/XTrader_20250115.log
```

### 2. 进程监控

```bash
# 查看进程
ps aux | grep XTrader

# 查看 CPU/内存
top -p $(pgrep XTrader)

# 查看线程
ps -eLf | grep XTrader
```

### 3. 共享内存检查

```bash
# 查看共享内存
ipcs -m

# 查看特定共享内存
ipcs -m | grep MarketServer

# 清理共享内存
ipcrm -M <key>
```

### 4. 网络检查

```bash
# 查看网络连接
netstat -anp | grep XTrader

# 测试柜台连通性
telnet 180.168.146.187 10130

# 查看网络流量
iftop -i eth0
```

## 常见问题排查

### 1. 编译错误

**问题**: YAML-CPP 链接错误
```
undefined reference to `YAML::LoadFile'
```

**解决**:
```bash
cd XAPI/YAML-CPP/0.8.0/
mkdir build && cd build
cmake .. -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=0"
make
```

### 2. 运行时错误

**问题**: 连接柜台失败
```
OnRspUserLogin failed, ErrorID=-1
```

**排查**:
1. 检查网络: `ping 180.168.146.187`
2. 检查端口: `telnet 180.168.146.187 10130`
3. 检查配置: `cat config/CTP.yml`
4. 查看 API 版本是否匹配

**问题**: 共享内存连接失败
```
SHM connect failed
```

**排查**:
1. 检查是否先启动服务端 (XTrader/XMarketCenter)
2. 检查共享内存: `ipcs -m`
3. 清理共享内存: `ipcrm -M <key>`
4. 检查共享内存权限

### 3. 性能问题

**问题**: 延迟过高

**排查**:
1. 检查 CPU 绑定: `taskset -pc <pid>`
2. 检查 CPU 隔离: `cat /proc/cmdline`
3. 检查 CPU 频率: `cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq`
4. 查看日志延迟统计

## 停止流程

### 优雅停止

```bash
# 停止策略
pkill XQuant

# 停止交易
pkill XTrader

# 停止风控
pkill XRiskJudge

# 停止行情
pkill XMarketCenter

# 停止监控
pkill XWatcher
```

### 强制停止

```bash
pkill -9 XQuant
pkill -9 XTrader
pkill -9 XRiskJudge
pkill -9 XMarketCenter
pkill -9 XWatcher
```

### 清理共享内存

```bash
# 删除所有共享内存
ipcs -m | awk '{print $2}' | xargs -I {} ipcrm -m {}
```

## 备份与恢复

### 备份

```bash
# 备份配置
tar -czf config_backup_$(date +%Y%m%d).tar.gz config/

# 备份日志
tar -czf logs_backup_$(date +%Y%m%d).tar.gz logs/

# 备份数据库
cp XServer.db XServer.db.bak
```

### 恢复

```bash
# 恢复配置
tar -xzf config_backup_20250115.tar.gz

# 恢复数据库
cp XServer.db.bak XServer.db
```

## 安全建议

1. **网络隔离**: Colo 服务器与用户侧通过专线或 VPN 连接
2. **权限控制**: 使用普通用户运行交易程序，避免 root
3. **密码管理**: 配置文件中的密码加密存储
4. **日志审计**: 定期检查日志，监控异常操作
5. **防火墙**: 配置防火墙规则，仅开放必要端口

## 升级更新

### 升级流程

1. **备份当前版本**:
```bash
cp -r QuantFabric QuantFabric.bak
```

2. **拉取最新代码**:
```bash
cd QuantFabric
git pull
git submodule update --remote
```

3. **重新编译**:
```bash
sh build_clear.sh
sh build_release.sh
```

4. **停止旧版本**:
```bash
./stop_trading.sh
```

5. **启动新版本**:
```bash
./start_trading.sh
```

6. **验证**:
- 检查日志
- 测试行情接收
- 测试报单功能

## 联系支持

**QQ 群**: 748930268 (验证码: QuantFabric)
**GitHub**: https://github.com/QuantFabric/QuantFabric
