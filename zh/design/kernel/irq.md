# 中断与异常处理设计

## 1. 概述

在 Glenda 中，内核在中断和异常处理中的作用很小。它将硬件事件转换为 IPC 消息，允许用户空间线程根据自己的策略处理它们。

## 2. 异常处理 (应用程序中断)

内核不在内部处理用户异常（缺页异常、非法指令等）。这些被视为 **应用程序中断**。

### 2.1 工作流
1.  **Trap 入口**: 内核捕获异常。
2.  **状态保存**: 将故障详细信息 (`scause`, `stval`, `sepc`) 保存到线程的 UTCB 中。
3.  **IPC 生成**: 内核构造“Fault Message”。
4.  **IPC 发送**: 内核强制故障线程将此消息发送到其注册的 **ExceptionHandler Endpoint**（存储在 TCB 的 `irqhandler` 字段中）。
5.  **阻塞**: 线程进入 `BlockedSend` 状态。
6.  **解决**: ExceptionHandler（用户空间）接收消息，修复问题（例如，映射页面），并回复（通过 `sys_send` 到故障线程的 reply cap 或类似机制，或通过调用 `TCB::Resume`）。
7.  **恢复**: 线程在故障指令（或下一条，取决于回复）处恢复执行。

## 3. 中断处理 (用户模式驱动程序)

内核驱动程序被移除。硬件中断必须通过动态分发机制到达用户空间驱动程序。

### 3.1 IRQ 对象
*   **`IrqHandler` Capability**: 代表管理特定硬件中断线（例如 UART 的 IRQ 10）的权限。
*   **`Endpoint` 对象**: 用于接收中断通知的标准 IPC 端点。

### 3.2 注册流程
1.  **启动**: Root Task 接收所有系统中断的 `IrqHandler` capabilities。
2.  **驱动启动**: Root Task 启动驱动程序（例如 UART 驱动程序）。
3.  **委托**: Root Task 将特定的 `IrqHandler`（例如 IRQ 10）授予 UART 驱动程序。
4.  **绑定**:
    *   UART 驱动程序创建 `Endpoint` 或 `Notification` 对象。
    *   UART 驱动程序调用 `IrqHandler.SetNotification(Cap)`。
    *   内核在全局 `IRQ_TABLE` 中记录此映射。

### 3.3 处理流程
1.  **硬件**: 中断触发。
2.  **内核 (Trap)**:
    *   **认领**: 内核从 PLIC 认领 IRQ。
    *   **屏蔽**: 内核在 PLIC 处屏蔽中断以防止中断风暴。
    *   **通知**: 内核向绑定的 `Endpoint` 发送非阻塞信号/消息。
    *   **完成**: 内核向 PLIC 发出“完成”信号（但线路保持屏蔽）。
3.  **用户空间 (驱动程序)**:
    *   线程从 `sys_recv` 唤醒。
    *   处理硬件（例如，读取 UART RX FIFO）。
    *   调用 `IrqHandler.Ack()`。
4.  **内核 (Syscall)**:
    *   **取消屏蔽**: 内核在 PLIC 处取消屏蔽中断，允许其再次触发。

## 4. 数据结构

### 4.1 IrqSlot
表示硬件 IRQ 线的内核内部结构。内核维护这些槽的静态数组 `IRQ_TABLE`（例如 `[IrqSlot; 64]`）。

```rust
struct IrqSlot {
    notification: Option<Capability>, // 绑定的 Endpoint/Notification
    enabled: bool,                    // 当前屏蔽状态
}
```

### 4.2 IrqHandler Capability
面向用户的 IRQ 句柄。
*   **对象**: `CapType::IrqHandler { irq: usize }`
*   **方法**:
    *   `SetNotification(cap)`: 将 IPC 对象绑定到此 IRQ。
    *   `Ack()`: 处理后取消屏蔽 IRQ。
    *   `Clear()`: 解除绑定通知。
