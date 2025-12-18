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
    *   The UART Driver creates an `Endpoint` object.
    *   The UART Driver invokes `IrqHandler.SetEndpoint(EndpointCap)`.
    *   The kernel records this mapping: `IRQ 10 -> Endpoint A`.

### 3.3 Handling Flow
1.  **Hardware**: Interrupt fires.
2.  **Kernel**:
    *   Masks the interrupt at the controller (PLIC/APIC).
    *   Looks up the bound `Endpoint` object.
    *   Sends a message to the `Endpoint` (non-blocking or special kernel message).
3.  **User Space (Driver)**:
    *   Thread wakes up from `sys_recv(Endpoint)`.
    *   Reads device registers, processes data.
    *   Invokes `IrqHandler.Ack()` to unmask the interrupt.
