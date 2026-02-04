# 交互协议设计 (Interaction Protocols)

## 1. 简介
本部分定义了 Glenda 内部使用的通信协议，对应 `libglenda-rs/ipc/proto` 中的定义。

## 2. 协议分层

Glenda 采用分层协议架构：

### 2.1 第 0 层：微内核 IPC (Raw)
*   **机制**: seL4/微内核 `Call`/`Reply`。
*   **传输**: CPU 寄存器 + UTCB (消息寄存器)。
*   **MsgTag**: 首个寄存器，包含协议 ID (Label) 和参数计数。

### 2.2 第 1 层：传输协议

#### A. 9P2000 (控制平面)
用于配置、命名空间管理和低带宽 I/O。
*   **封装**: 9P 消息被打包进 UTCB。如果消息超过 UTCB 大小，以前使用共享内存。
*   **消息类型**: `Tattach`, `Twalk`, `Topen`, `Tread`, `Twrite`, `Tclunk`。

#### B. 环形缓冲区 (数据平面)
用于高性能数据传输（例如网络数据包、大容量磁盘 I/O）。
*   **结构**: 包含描述符环和数据缓冲区的共享内存区域。
*   **信号**: 仅在需要唤醒对端时发送“门铃” IPC (Notifications)。

### 2.3 第 2 层：服务协议

针对核心服务特定定义的指令集，通过唯一的 **协议标签 (Protocol Labels)** 区分。详情请参阅各单独文件。

| 协议 | Label 范围 | 描述 | 参考 |
| :--- | :--- | :--- | :--- |
| **Generic** | `0x000` | 通用回复 | [generic.md](generic.md) |
| **Kernel** | `0x100 - 0x1FF` | 内核核心操作 | [kernel.md](kernel.md) |
| **Process** (Factotum) | `0x200 - 0x2FF` | 进程生命周期 | [process.md](process.md) |
| **Device** (Unicorn) | `0x300 - 0x3FF` | 设备管理 | [device.md](device.md) |
| **Init** | `0x400 - 0x4FF` | 启动协议 | [init.md](init.md) |
| **Fossil** | `0x500 - 0x5FF` | 文件系统控制 | [fs.md](fs.md) |
| **Gopher** | `0x600 - 0x6FF` | 网络栈控制 | [net.md](net.md) |
| **Chimera** | `0x700 - 0x7FF` | 虚拟化管理 | [virt.md](virt.md) |

## 3. 数据定义 (`libglenda-rs/ipc/proto`)

有关结构体定义和指令 ID，请参阅特定的协议文档。

### 3.1 通用类型
定义了系统中使用的 `Fid` (文件 ID), `Qid` (唯一 ID), `Stat` 结构体。
