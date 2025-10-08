# 常见问题 (FAQ)

## 编译相关

### Q: 编译时出现 "undefined reference to ..." 错误？

**A**: 检查以下几点：
1. 确认使用 `-D_GLIBCXX_USE_CXX11_ABI=0` 编译选项
2. 检查链接库路径是否正确
3. 确认所有依赖库已正确安装

```bash
# 清理重新编译
rm -rf build
sh build_release.sh
```

### Q: 找不到 YAML-CPP 或其他第三方库？

**A**: 检查 XAPI 子模块是否正确初始化：
```bash
git submodule update --init --recursive
```

### Q: Ubuntu 编译失败，提示缺少 libstdc++？

**A**: 参考 README 中的 Ubuntu 编译说明，安装必要的依赖：
```bash
sudo apt-get install build-essential cmake libstdc++6
```

## 运行相关

### Q: 启动时提示 "Shared memory create failed"？

**A**: 可能的原因：
1. 共享内存未清理：
   ```bash
   ipcs -m | grep <key>
   ipcrm -m <shmid>
   ```
2. 权限不足：使用 `sudo` 或调整权限
3. 系统共享内存限制：修改 `/etc/sysctl.conf`
   ```bash
   kernel.shmmax = 68719476736  # 64GB
   kernel.shmall = 4294967296
   sudo sysctl -p
   ```

### Q: XTrader 报 "Load plugin failed"？

**A**: 检查：
1. 插件文件路径是否正确
2. 插件是否导出 `CreatePlugin` 符号：
   ```bash
   nm -D libCTPTradeGateWay.so | grep CreatePlugin
   ```
3. 插件依赖的柜台 API 库是否存在

### Q: 连接柜台失败？

**A**: 排查步骤：
1. 检查网络连接：
   ```bash
   ping <柜台IP>
   telnet <柜台IP> <柜台端口>
   ```
2. 检查防火墙规则
3. 确认账户、密码、BrokerID 等配置正确
4. 查看柜台是否在维护时间

### Q: 报单被拒，错误码 -3？

**A**: CTP 错误码 -3 表示：
- 合约代码错误
- 交易所代码错误
- 合约未订阅

检查合约代码和交易所代码是否正确。

## 性能相关

### Q: Tick2Order 延迟很高（>100μs）？

**A**: 性能优化建议：
1. **CPU 绑定**：在配置文件中设置 CPUSET
2. **CPU 超频**：提升 CPU 主频到 4.0GHz+
3. **关闭超线程**：BIOS 中禁用 Hyper-Threading
4. **CPU 隔离**：
   ```bash
   # /etc/default/grub
   GRUB_CMDLINE_LINUX="isolcpus=10,11,12,13"
   sudo update-grub
   sudo reboot
   ```
5. **使用 Onload**：网络内核旁路加速

### Q: 共享内存通信延迟高？

**A**: 检查：
1. 是否使用 CPU 绑定
2. 是否关闭 CPU 节能模式：
   ```bash
   sudo cpupower frequency-set -g performance
   ```
3. 检查是否有其他进程竞争 CPU

### Q: 策略吞吐量不足？

**A**:
1. 使用 C++ 策略替代 Python
2. 优化策略逻辑，减少计算量
3. 使用批量处理
4. 考虑使用 HFTrader 单进程架构

## 配置相关

### Q: 如何配置多账户？

**A**: 启动多个 XTrader 实例，每个实例配置不同账户：
```bash
./XTrader_0.9.3 XTrader_Account1.yml libCTPTradeGateWay.so &
./XTrader_0.9.3 XTrader_Account2.yml libCTPTradeGateWay.so &
```

### Q: 如何配置多 Colo？

**A**: 在 XServer 配置中添加多个 Colo 配置，每个 Colo 运行独立的 XWatcher。

### Q: 风控参数如何调整？

**A**: 编辑 XRiskJudge 配置文件：
```yaml
RiskConfig:
  FlowLimit: 100        # 流速限制（笔/秒）
  CancelRatio: 0.5      # 撤单比例
  SelfTradeCheck: true  # 自成交检查
```

## 调试相关

### Q: 如何查看详细日志？

**A**:
1. 日志文件位置：`./logs/<AppName>_YYYYMMDD.log`
2. 调整日志级别（在代码中）：
   ```cpp
   Logger::SetLevel(spdlog::level::debug);
   ```
3. 使用 `tail -f` 实时查看：
   ```bash
   tail -f logs/XTrader_20250108.log
   ```

### Q: 如何使用 GDB 调试？

**A**:
```bash
gdb --args ./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so

(gdb) break TraderEngine::Run
(gdb) run
(gdb) bt
(gdb) print variable
```

### Q: 如何检测内存泄漏？

**A**: 使用 Valgrind：
```bash
valgrind --leak-check=full --show-leak-kinds=all \
  ./XTrader_0.9.3 XTrader.yml libCTPTradeGateWay.so
```

## 部署相关

### Q: 如何部署到生产环境？

**A**: 参考 [部署运维指南](deployment/DeploymentGuide.md)：
1. 环境准备（依赖安装）
2. 编译部署（Release 模式）
3. 配置文件（生产配置）
4. 系统优化（CPU/网络/内存）
5. 启动脚本（自动启动）
6. 监控告警（性能监控）

### Q: 如何实现高可用？

**A**:
1. **双 Colo 部署**：主备 Colo，自动切换
2. **进程守护**：使用 systemd 或 supervisor
3. **异常恢复**：自动重连、订单恢复
4. **数据备份**：订单日志、持仓快照

### Q: 如何升级系统？

**A**:
1. 停止所有服务
2. 备份配置文件和数据
3. 编译新版本
4. 替换二进制文件
5. 验证配置兼容性
6. 启动新版本
7. 验证功能

## 开发相关

### Q: 如何开发新的行情插件？

**A**: 参考 [XMarketCenter 插件开发指南](modules/XMarketCenter/03_PluginDevelopment.md)

### Q: 如何开发新的交易插件？

**A**: 参考 [XTrader 插件开发指南](modules/XTrader/03_PluginDevelopment.md)

### Q: 如何开发 C++ 策略？

**A**:
1. 继承 `Strategy` 基类
2. 实现 `OnTick`、`OnOrder` 等回调
3. 编译成可执行文件或动态库
4. 配置启动

示例代码参考 XQuant 模块文档。

### Q: 如何开发 Python 策略？

**A**:
1. 使用 SHMServer 的 Python 绑定
2. 实现策略逻辑
3. 订阅行情和订单回报
4. 发送报单

## 其他问题

### Q: 系统支持哪些柜台？

**A**:
- **期货**: CTP、REM、YD
- **股票**: OES、XTP、Tora
- 可通过插件扩展其他柜台

### Q: 系统支持哪些操作系统？

**A**:
- **主要支持**: CentOS 7/8、Ubuntu 18.04/20.04
- **其他 Linux**: 需要适配编译
- **Windows**: 不支持（部分组件可移植）

### Q: 商业版和开源版有什么区别？

**A**:
- **开源版（QuantFabric）**: 多进程架构，Tick2Order 20-60μs
- **商业版（HFTrader）**: 单进程架构，Tick2Order < 2μs，支持 YD 易达

### Q: 如何获取技术支持？

**A**:
- **QQ 群**: 748930268 (验证码: QuantFabric)
- **GitHub Issues**: https://github.com/QuantFabric/QuantFabric/issues
- **邮件**: 联系作者 @yutiansut

### Q: 如何贡献代码？

**A**:
1. Fork 仓库
2. 创建功能分支
3. 提交代码
4. 发起 Pull Request
5. 代码审查
6. 合并到主分支

---

**未解决的问题？**

请在 [GitHub Issues](https://github.com/QuantFabric/QuantFabric/issues) 提出，或加入 QQ 群交流。

---
作者: @scorpiostudio @yutiansut
