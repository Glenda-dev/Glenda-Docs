# Interrupt & Exception Handling Migration Guide

## 1. Current Status
Currently, `kernel/src/irq/trap/user.rs` handles traps directly:
- **Syscalls**: Dispatches to `syscall::dispatch`.
- **Exceptions**: Panics or kills the process.
- **Interrupts**: Handled by kernel drivers.

## 2. Target Architecture (Microkernel)

### 2.1 Exception Handling (Application Interrupts)
The kernel must not handle user exceptions (Page Fault, Illegal Instruction, etc.) internally. These are treated as **Application Interrupts**.

**New Workflow:**
1.  **Trap Entry**: Kernel catches exception.
2.  **State Save**: Save fault details (`scause`, `stval`, `sepc`) into the thread's UTCB.
3.  **IPC Generation**: Kernel constructs a "Fault Message".
4.  **IPC Send**: Kernel forces the faulting thread to send this message to its registered **ExceptionHandler Endpoint** (stored in the TCB's `irqhandler` field).
5.  **Block**: The thread enters `BlockedException` state.
6.  **Resolution**: The ExceptionHandler (user-space) receives the message, fixes the issue (e.g., maps a page), and replies.
7.  **Resume**: The thread resumes execution at the faulting instruction (or next, depending on reply).

### 2.2 Interrupt Handling (User-Mode Drivers)
Kernel drivers are removed. Hardware interrupts must reach user-space drivers via a dynamic dispatch mechanism.

### 2.2.1 IRQ Objects
*   **`IrqHandler` Capability**: Represents the authority to manage a specific hardware interrupt line (e.g., IRQ 10 for UART).
*   **`Notification` Object**: An asynchronous synchronization primitive (binary or counting semaphore) used to wake up threads.

### 2.2.2 Registration Process
1.  **Startup**: The Root Task receives `IrqHandler` capabilities for all system interrupts.
2.  **Driver Startup**: The Root Task spawns a driver (e.g., UART Driver).
3.  **Delegation**: The Root Task grants the specific `IrqHandler` (e.g., IRQ 10) to the UART Driver.
4.  **Binding**:
    *   The UART Driver creates a `Notification` object.
    *   The UART Driver invokes `IrqHandler.SetNotification(NotificationCap)`.
    *   The kernel records this mapping: `IRQ 10 -> Notification A`.

### 2.2.3 Handling Flow
1.  **Hardware**: Interrupt fires.
2.  **Kernel**:
    *   Masks the interrupt at the controller (PLIC/APIC).
    *   Looks up the bound `Notification` object.
    *   Signals the `Notification` (sets a bit, wakes up waiting thread).
3.  **User Space (Driver)**:
    *   Thread wakes up from `sys_recv(Notification)`.
    *   Reads device registers, processes data.
    *   Invokes `IrqHandler.Ack()` to unmask the interrupt.

### 2.3 System Calls
- Only `ecall` triggers the IPC fast-path.
- All legacy syscall logic (`fs`, `proc`) is removed from the trap handler.
