# System Call Interface Migration Guide

## 1. Current Status
`kernel/src/syscall/mod.rs` implements a monolithic API:
- `SYS_FORK`, `SYS_EXIT`, `SYS_WAIT`
- `SYS_READ`, `SYS_WRITE`, `SYS_OPEN`
- `SYS_MMAP`, `SYS_BRK`

## 2. Target Architecture (Microkernel)
All specific functionality is moved to user space. The kernel API is generic.

### 2.1 New System Calls
The syscall table should be replaced with ~5-7 primitives:

| Syscall | Arguments | Description |
| :--- | :--- | :--- |
| **`sys_call`** | `cptr`, `msg_info`, `...` | Atomic Send + Recv. Used for RPC. |
| **`sys_reply_recv`** | `cptr`, `msg_info`, `...` | Reply to last caller and wait for next. (Server loop) |
| **`sys_send`** | `cptr`, `msg_info`, `...` | Send a message (blocking or non-blocking). |
| **`sys_recv`** | `cptr`, `out_msg`, `...` | Wait for a message. |
| **`sys_yield`** | `None` | Give up remaining timeslice. |
| **`sys_debug_putc`** | `char` | (Debug only) Print to serial console. |

### 2.2 Invocation Model
Most operations are performed by "invoking" a capability.
- **`sys_call(cptr, ...)`** is the generic entry point.
- If `cptr` is an **Endpoint**: Perform IPC.
- If `cptr` is a **Kernel Object** (e.g., PageTable, TCB): Perform a method on that object.
    - Example: `sys_call(page_table_cap, MAP_METHOD, frame_cap, vaddr, flags)`

### 2.3 Migration Steps
1.  Delete contents of `kernel/src/syscall/`.
2.  Create `kernel/src/syscall/ipc.rs` for IPC logic.
3.  Create `kernel/src/syscall/invocation.rs` for object method dispatch.
4.  Update `trap_user_handler` to decode registers into an `IPCMessage` struct.
