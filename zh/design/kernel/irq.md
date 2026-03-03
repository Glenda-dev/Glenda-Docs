# 中断处理设计

## 1. 概述

在 Glenda 中，内核在中断处理中的作用很小。它将硬件事件转换为 IPC 消息，允许用户空间线程根据自己的策略处理它们。

## 2. 外设中断处理

内核仅实现必要的驱动程序（中断控制器，调试输出）

### 2.1 IRQ 对象
*   **`IrqHandler` Capability**: 代表管理特定硬件中断线（例如 UART 的 IRQ 10）的权限。
*   **`Endpoint` 对象**: 用于接收中断通知的标准 IPC 端点。

### 2.2 注册流程
1.  **启动**: Root Task 接收系统中断的根 `IrqHandler` capabilitie。
2.  **驱动启动**: Root Task 启动驱动程序（例如 UART 驱动程序）。
3.  **委托**: Root Task 将特定的 `IrqHandler`（例如 IRQ 10）授予 UART 驱动程序。
4.  **绑定**:
    *   UART 驱动程序创建 `Endpoint` 或对象。
    *   UART 驱动程序调用 `IrqHandler.SetNotification(Cap)`。
    *   内核在全局 `IRQ_TABLE` 中记录此映射。

### 2.3 处理流程
1.  **硬件**: 中断触发。
2.  **内核 (Trap)**:
    *   **认领**: 内核从中断控制器认领 IRQ。
    *   **屏蔽**: 内核在中断控制器处屏蔽中断以防止中断风暴。
    *   **通知**: 内核向绑定的 `Endpoint` 发送非阻塞信号/消息。
    *   **完成**: 内核向中断控制器发出“完成”信号（但线路保持屏蔽）。
3.  **用户空间 (驱动程序)**:
    *   线程从 `sys_recv` 唤醒。
    *   处理硬件（例如，读取 UART RX FIFO）。
    *   调用 `IrqHandler.Ack()`。
4.  **内核 (Syscall)**:
    *   **取消屏蔽**: 内核在中断控制器处取消屏蔽中断，允许其再次触发。

## 3. 时间中断处理

内核采用了Tickless设计，动态计算下一个中断时间并设置触发，同时提供了应用程序自定义闹钟的功能

### 3.1 Tickless设计
1.  **计算下一个Tick**: 内核根据进程剩余时间片与闹钟，计算下一个中断触发时间，以毫秒计算
2.  **设置Tick**: 内核将下一个中断时间使用HAL层提供的接口设置，单位为毫秒

### 3.2 闹钟设计
内核将应用提供的 Endpoint 绑定到闹钟上，触发时则向绑定的端点发送通知

## 4. 数据结构

### 4.1 IrqSlot
表示硬件 IRQ 线的内核内部结构。内核维护这些槽的静态数组 `IRQ_TABLE`（例如 `[IrqSlot; 64]`）。

```rust
struct IrqSlot {
    notification: Option<Capability>, // 绑定的 Endpoint
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

### 4.3 Alarm
用于设定闹钟

```rust
pub struct Alarm {
    time: usize,   // 闹钟触发的时间点
    ep: Capability, // 绑定的 Endpoint
}
```