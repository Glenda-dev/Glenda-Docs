# Trap Subsystem Design

## 1. Overview

The Trap subsystem is responsible for handling all transitions between user-space and kernel-space, as well as managing hardware-level events. In Glenda's microkernel architecture, we strictly distinguish between:

- **Interrupts (Asynchronous)**: External hardware events (Timer, PLIC) managed by the `irq` module.
- **Exceptions (Synchronous)**: Events triggered by instruction execution (Page Fault, Illegal Instruction, Syscall) managed by the `trap` module.

## 2. Exception Handling (Fault IPC)

Glenda follows the "Policy-out-of-Kernel" principle. The kernel does not resolve exceptions like Page Faults internally. Instead, it converts them into **Fault IPC** messages.

### 2.1 The Fault Handler
Each thread (`TCB`) can have a registered **Fault Handler**, which is a capability to an IPC `Endpoint`. 

- If a fault occurs and a handler is registered, the kernel forces the thread to send a message to that endpoint.
- If no handler is registered, the kernel terminates the process or panics (for kernel-level faults).

### 2.2 Workflow
1.  **Trap Entry**: The CPU triggers a trap; the kernel saves the register context.
2.  **State Capture**: The kernel reads `scause`, `stval`, and `sepc`.
3.  **UTCB Update**: These values are written into the faulting thread's UTCB (IPC Buffer).
4.  **Message Construction**: A special "Fault Message" tag is created (e.g., Label `0xFFFF`).
5.  **Synchronous Send**: The kernel invokes the IPC subsystem to send the message to the `fault_handler`.
6.  **Blocking**: The faulting thread enters a `Blocked` state, waiting for a reply.
7.  **Resolution**: A user-space manager (e.g., a pager or debugger) receives the IPC, performs necessary actions (like mapping a page), and replies to the thread.
8.  **Resume**: Upon receiving the reply, the kernel restores the thread's state and resumes execution.

## 3. System Calls

System calls are a special type of exception (`Environment Call from U-mode`). 

- The `trap` module identifies the syscall.
- It dispatches the call to the `syscall` module.
- The result is placed in the `a0` register of the saved context before returning to user-space.

## 4. Data Structures

### 4.1 TCB Integration
```rust
pub struct TCB {
    // ...
    pub fault_handler: Option<Capability>, // Capability to an Endpoint
    pub ipc_buffer: VirtAddr,              // Pointer to UTCB
    // ...
}
```

### 4.2 Fault Message Format (in UTCB)
| Offset | Content | Description |
|--------|---------|-------------|
| 0      | scause  | Exception cause bits |
| 8      | stval   | Faulting address or instruction |
| 16     | sepc    | Program counter where fault occurred |
