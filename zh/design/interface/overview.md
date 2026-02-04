# 开发接口设计 (Development Interfaces)

## 1. 简介
本部分描述了 `libglenda-rs/interface` 为应用程序和服务开发者提供的高级 Rust API 和 Trait。

## 2. 接口模块

| 模块 | 描述 | 参考 |
| :--- | :--- | :--- |
| **Process** | 进程生命周期、内存和故障 | [process.md](process.md) |
| **Device** | 硬件、PCI、DMA 访问 | [device.md](device.md) |
| **Resource** | 内核对象管理 (VSpace, CSpace) | [resource.md](resource.md) |
| **Server** | 系统服务框架 | [server.md](server.md) |
| **Async** | Async/Await 支持 | [async.md](async.md) |

## 3. 核心 Trait 摘要

接口层将底层的 IPC 机制抽象为符合 Rust 惯例的 Trait。请参阅详细文件以获取完整 API 列表。
