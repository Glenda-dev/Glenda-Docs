# Interrupt & Exception Handling Design

## 1. Overview

In Glenda, the kernel's role in interrupt and exception handling is minimal. It converts hardware events into IPC messages, allowing user-space threads to handle them according to their own policies.

## 2. Exception Handling (Application Interrupts)

The kernel does not handle user exceptions (Page Fault, Illegal Instruction, etc.) internally. These are treated as **Application Interrupts**.

### 2.1 Workflow
1.  **Trap Entry**: Kernel catches exception.
2.  **State Save**: Save fault details (`scause`, `stval`, `sepc`) into the thread's UTCB.
3.  **IPC Generation**: Kernel constructs a "Fault Message".
4.  **IPC Send**: Kernel forces the faulting thread to send this message to its registered **ExceptionHandler Endpoint** (stored in the TCB's `irqhandler` field).
5.  **Block**: The thread enters `BlockedSend` state.
6.  **Resolution**: The ExceptionHandler (user-space) receives the message, fixes the issue (e.g., maps a page), and replies (via `sys_send` to the faulting thread's reply cap or similar mechanism, or by invoking `TCB::Resume`).
7.  **Resume**: The thread resumes execution at the faulting instruction (or next, depending on reply).

## 3. Interrupt Handling (User-Mode Drivers)

Kernel drivers are removed. Hardware interrupts must reach user-space drivers via a dynamic dispatch mechanism.

### 3.1 IRQ Objects
*   **`IrqHandler` Capability**: Represents the authority to manage a specific hardware interrupt line (e.g., IRQ 10 for UART).
*   **`Endpoint` Object**: A standard IPC endpoint used to receive interrupt notifications.

### 3.2 Registration Process
1.  **Startup**: The Root Task receives `IrqHandler` capabilities for all system interrupts.
2.  **Driver Startup**: The Root Task spawns a driver (e.g., UART Driver).
3.  **Delegation**: The Root Task grants the specific `IrqHandler` (e.g., IRQ 10) to the UART Driver.
4.  **Binding**:
    *   The UART Driver creates an `Endpoint` or `Notification` object.
    *   The UART Driver invokes `IrqHandler.SetNotification(Cap)`.
    *   The kernel records this mapping in a global `IRQ_TABLE`.

### 3.3 Handling Flow
1.  **Hardware**: Interrupt fires.
2.  **Kernel (Trap)**:
    *   **Claim**: Kernel claims the IRQ from PLIC.
    *   **Mask**: Kernel masks the interrupt at the PLIC to prevent interrupt storms.
    *   **Notify**: Kernel sends a non-blocking signal/message to the bound `Endpoint`.
    *   **Complete**: Kernel signals "Complete" to PLIC (but the line remains masked).
3.  **User Space (Driver)**:
    *   Thread wakes up from `sys_recv`.
    *   Processes hardware (e.g., reads UART RX FIFO).
    *   Invokes `IrqHandler.Ack()`.
4.  **Kernel (Syscall)**:
    *   **Unmask**: Kernel unmasks the interrupt at the PLIC, allowing it to fire again.

## 4. Data Structures

### 4.1 IrqSlot
A kernel-internal structure representing a hardware IRQ line.
```rust
struct IrqSlot {
    notification: Option<Capability>, // Bound Endpoint/Notification
    enabled: bool,                    // Current mask state
}
```

### 4.2 IrqHandler Capability
The user-facing handle for an IRQ.
*   **Object**: `CapType::IrqHandler { irq: usize }`
*   **Methods**:
    *   `SetNotification(cap)`: Bind an IPC object to this IRQ.
    *   `Ack()`: Unmask the IRQ after processing.
    *   `Clear()`: Unbind the notification.
