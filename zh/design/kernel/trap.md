# Trap 子系统设计

## 1. 概述

Trap 子系统负责处理用户空间和内核空间之间的所有转换，以及管理硬件级事件。在 Glenda 的微内核架构中，我们严格区分：

- **中断 (异步)**：由 `irq` 模块管理的外部硬件事件（定时器，PLIC）。
- **异常 (同步)**：由 `trap` 模块管理的指令执行触发的事件（缺页异常，非法指令，系统调用）。

## 2. 异常处理 (Fault IPC)

Glenda 遵循“策略在内核之外”的原则。内核不在内部解决像缺页异常这样的异常。相反，它将它们转换为 **Fault IPC** 消息。

### 2.1 Fault Handler
每个线程 (`TCB`) 都可以有一个注册的 **Fault Handler**，它是一个指向 IPC `Endpoint` 的 capability。

- 如果发生故障且注册了处理程序，内核强制线程向该端点发送消息。
- 如果未注册处理程序，内核将终止进程或 panic（对于内核级故障）。

### 2.2 工作流
1.  **Trap 入口**: CPU 触发 trap；内核保存寄存器上下文。
2.  **状态捕获**: 内核读取 `scause`, `stval`, 和 `sepc`。
3.  **UTCB 更新**: 这些值被写入故障线程的 UTCB (IPC 缓冲区)。
4.  **消息构造**: 创建一个特殊的“Fault Message”标签（例如，Label `0xFFFF`）。
5.  **同步发送**: 内核调用 IPC 子系统将消息发送到 `fault_handler`。
6.  **阻塞**: 故障线程进入 `Blocked` 状态，等待回复。
7.  **解决**: 用户空间管理器（例如 pager 或调试器）接收 IPC，执行必要的操作（如映射页面），并回复线程。
8.  **恢复**: 收到回复后，内核恢复线程状态并继续执行。

## 3. 系统调用

系统调用是一种特殊类型的异常 (`Environment Call from U-mode`)。

- `trap` 模块识别系统调用。
- 它将调用分派给 `syscall` 模块。
- 结果在返回用户空间之前放置在保存的上下文的 `a0` 寄存器中。

## 4. 数据结构

### 4.1 TCB 集成
```rust
pub struct TCB {
    // ...
    pub fault_handler: Option<Capability>, // 指向 Endpoint 的 Capability
    pub ipc_buffer: VirtAddr,              // 指向 UTCB 的指针
    // ...
}
```

### 4.2 Fault 消息格式 (在 UTCB 中)

当发生故障时，内核填充故障线程的 UTCB 如下：

| 字段 | 内容 | 描述 |
|-------|---------|-------------|
| `msg_tag` | `0xFFFF` | 指示 Fault IPC 的标签 |
| `mrs_regs[0]` | `scause` | 异常原因位 |
| `mrs_regs[1]` | `stval` | 故障地址或指令 |
| `mrs_regs[2]` | `sepc` | 发生故障的程序计数器 |

然后线程在它的 `fault_handler` 端点上进入 `BlockedSend` 状态。
