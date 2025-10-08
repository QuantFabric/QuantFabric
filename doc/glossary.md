# 术语表

## A

**API (Application Programming Interface)**
- 应用程序编程接口，用于不同软件组件之间的交互

**ABI (Application Binary Interface)**
- 应用程序二进制接口，定义二进制层级的兼容性

## C

**Colo (Co-Location)**
- 托管服务，将交易服务器放置在交易所附近以降低延迟

**CTP (Comprehensive Transaction Platform)**
- 上期技术综合交易平台，国内期货主流交易接口

**CPU Binding (CPU 绑定)**
- 将线程绑定到特定 CPU 核心，避免上下文切换

## D

**dlopen/dlsym**
- Linux 动态库加载函数，用于运行时加载共享库

## H

**HPSocket**
- 高性能网络通信框架

**HFTrader (High-Frequency Trader)**
- 高频交易系统，单进程架构版本

## I

**IPC (Inter-Process Communication)**
- 进程间通信，如共享内存、管道、Socket 等

## L

**Lock-Free Queue (无锁队列)**
- 使用原子操作实现的无互斥锁队列，降低延迟

## M

**Memory Order (内存序)**
- 多线程环境下内存操作的顺序保证
- relaxed: 无同步
- acquire: 获取其他线程的写入
- release: 释放写入，让其他线程可见

## O

**OES (Order Execution System)**
- 东方证券期权交易系统

**Onload**
- Solarflare 网络内核旁路技术，降低网络延迟

## P

**Pack 协议**
- 基于长度头的网络协议：4-byte 长度 + 数据体

**Plugin (插件)**
- 动态加载的功能模块

## R

**REM (Rapid Execution Mode)**
- 恒生 REM 柜台，低延迟交易接口

**RAII (Resource Acquisition Is Initialization)**
- C++ 资源管理技术，构造时获取资源，析构时释放

## S

**SHM (Shared Memory)**
- 共享内存，最快的 IPC 方式

**SPSC (Single Producer Single Consumer)**
- 单生产者单消费者队列模式

**SPDLog**
- 快速 C++ 日志库

## T

**Tick**
- 行情快照数据

**Tick2Order**
- 从接收行情到发送订单的延迟时间

**Ticker**
- 合约代码，如 rb2505

## V

**VDSO (Virtual Dynamic Shared Object)**
- Linux 内核提供的虚拟共享对象，用于加速系统调用

## X

**XTP (X-Trade Platform)**
- 中泰证券 XTP 交易平台

## Y

**YD (YI DA)**
- 易达柜台，纳秒级低延迟交易接口

**YAML**
- 人类可读的配置文件格式

## 其他

**零拷贝 (Zero-Copy)**
- 数据传输时避免内存拷贝，直接共享内存

**柜台**
- 券商/期货公司提供的交易接口系统

**开平仓**
- 期货交易中的开仓 (Open) 和平仓 (Close)

**投机套保**
- 期货交易类型：投机 (Speculation)、套利 (Arbitrage)、套保 (Hedge)

---
作者: @scorpiostudio @yutiansut
