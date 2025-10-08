# SHMServer 文档索引

## 📚 文档列表

### 1. [架构与原理](01_Architecture.md)
深入理解共享内存 IPC 技术

**内容**:
- 为什么需要共享内存 IPC
- 核心设计目标（极致低延迟、高可靠性、多语言支持）
- 整体架构设计
- 通信模式（发布-订阅、请求-响应）
- 内存布局详解
- 核心实现（SHMServer、SHMConnection）
- 性能优化（无锁算法、缓存优化、系统级优化）

**适合**:
- 系统架构师
- 性能优化工程师
- 想深入理解 IPC 的开发者

### 2. [基础文档 - 模块概览](../SHMServer.md)
快速了解 SHMServer

**内容**:
- 模块概述
- 核心功能
- Python 绑定使用
- 编译配置

**适合**:
- 初学者
- 快速参考

## 🎯 关键技术点

### 无锁队列 (SPSC)
- 单生产者单消费者模型
- 原子操作与内存序
- 环形缓冲区设计
- 缓存行对齐（避免伪共享）

### 共享内存
- System V vs POSIX
- 内存段创建与映射
- 进程间同步
- 异常恢复机制

### 性能优化
- 零拷贝数据传输
- 大页内存 (Huge Pages)
- 内存锁定 (mlock)
- CPU 缓存优化

## 📊 性能对比

| IPC 方式 | 延迟 | 带宽 |
|---------|------|------|
| Socket | 10-50μs | 中 |
| 管道 | 5-20μs | 低 |
| 消息队列 | 10-30μs | 中 |
| **SHMServer** | **1-5μs** | **高** |

## 📖 相关文档

- [XMarketCenter 技术实现](../XMarketCenter/02_TechnicalImplementation.md) - 共享内存使用示例
- [系统总体架构](../../architecture/SystemArchitecture.md) - IPC 在系统中的应用

## 🔗 参考资料

- Linux 共享内存编程: `man shm_overview`
- 原子操作与内存序: https://en.cppreference.com/w/cpp/atomic/memory_order
