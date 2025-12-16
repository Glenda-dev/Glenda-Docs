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
4.  **IPC Send**: Kernel forces the faulting thread to send this message to its registered **IRQHandler Endpoint**.
5.  **Block**: The thread enters `BlockedException` state.
6.  **Resolution**: The IRQHandler (user-space) receives the message, fixes the issue (e.g., maps a page), and replies.
7.  **Resume**: The thread resumes execution at the faulting instruction (or next, depending on reply).

### 2.2 Interrupt Handling (User-Mode Drivers)
Kernel drivers are removed. Hardware interrupts must reach user-space drivers.

**New Workflow:**
1.  **Registration**: Driver thread invokes an `IrqHandler` capability to bind a hardware IRQ to a **Notification Object**.
2.  **IRQ Firing**: Hardware triggers interrupt -> Kernel trap handler runs.
3.  **Masking**: Kernel masks the specific IRQ (to prevent storms).
4.  **Notification**: Kernel signals the Notification Object (sets a bit).
5.  **Ack**: Driver thread (waiting on the Notification) wakes up, handles the device, and invokes `IrqHandler` cap to unmask the IRQ.

### 2.3 System Calls
- Only `ecall` triggers the IPC fast-path.
- All legacy syscall logic (`fs`, `proc`) is removed from the trap handler.
