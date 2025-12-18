# System Call Interface Design

## 1. Overview

The Glenda microkernel exposes a minimal system call interface. All specific functionality (file I/O, process creation, etc.) is moved to user space and accessed via IPC. The kernel API is generic and capability-based.

## 2. System Calls

The syscall table consists of 3 primitives:

| Syscall | Arguments | Description |
| :--- | :--- | :--- |
| **`sys_invoke`** | `cptr`, `msg_info`, `...` | Invoke a method on a kernel object (TCB, PageTable, etc.). |
| **`sys_send`** | `cptr`, `msg_info`, `...` | Send a message to an Endpoint (blocking). |
| **`sys_recv`** | `cptr`, `out_msg`, `...` | Wait for a message from an Endpoint. |

## 3. Invocation Model

Most operations are performed by "invoking" a capability using `sys_invoke`.

*   **Mechanism**:
    *   The user provides a Capability Pointer (`cptr`) to a kernel object.
    *   The kernel looks up the object and performs the requested method.
*   **Arguments**:
    *   Primary arguments (Method ID) are passed in registers (e.g., `MR0`).
    *   Method-specific parameters are passed in registers and the **UTCB**.
*   **Results**:
    *   Status code returned in register.
    *   Method-specific results written to **UTCB**.

## 4. Kernel Object Methods

The following operations are defined for kernel objects.

### 4.1 Untyped
Used to allocate new kernel objects from free physical memory.

| Method | ID | Arguments | Description |
| :--- | :--- | :--- | :--- |
| **`Retype`** | 1 | `type`, `size_bits`, `root_cnode`, `node_index`, `node_depth`, `num_objects` | Create new objects. |

### 4.2 TCB (Thread Control Block)
Manage thread execution and state.

| Method | ID | Arguments | Description |
| :--- | :--- | :--- | :--- |
| **`Configure`** | 1 | `cspace_root`, `vspace_root`, `utcb_ptr`, `fault_ep` | Set critical thread properties. |
| **`SetPriority`** | 2 | `priority` | Set scheduling priority. |
| **`SetRegisters`** | 3 | `flags`, `arch_flags`, `regs...` | Write to thread context. |
| **`GetRegisters`** | 4 | `flags`, `arch_flags` | Read from thread context. |
| **`Resume`** | 5 | - | Make thread runnable. |
| **`Suspend`** | 6 | - | Suspend thread execution. |

### 4.3 PageTable (VSpace)
Manage virtual address spaces.

| Method | ID | Arguments | Description |
| :--- | :--- | :--- | :--- |
| **`Map`** | 1 | `frame_cap`, `vaddr`, `flags` | Map a frame into this page table. |
| **`Unmap`** | 2 | `vaddr` | Unmap a page or page table. |

### 4.4 CNode (Capability Space)
Manage capabilities (Copy, Move, Mint, Revoke). These methods are invoked on a **CNode** capability to manipulate slots within it (or relative to it).

| Method | ID | Arguments | Description |
| :--- | :--- | :--- | :--- |
| **`Copy`** | 1 | `dest_index`, `dest_depth`, `src_cnode`, `src_index`, `src_depth`, `rights` | Copy a capability. |
| **`Mint`** | 2 | `dest_index`, `dest_depth`, `src_cnode`, `src_index`, `src_depth`, `rights`, `badge` | Copy with badge/reduced rights. |
| **`Move`** | 3 | `dest_index`, `dest_depth`, `src_cnode`, `src_index`, `src_depth` | Move a capability. |
| **`Mutate`** | 4 | `dest_index`, `dest_depth`, `src_cnode`, `src_index`, `src_depth`, `badge` | Move with badge (no copy). |
| **`Revoke`** | 5 | `index`, `depth` | Delete all children of cap at index. |
| **`Delete`** | 6 | `index`, `depth` | Delete cap at index. |

### 4.5 IrqHandler
Manage hardware interrupts.

| Method | ID | Arguments | Description |
| :--- | :--- | :--- | :--- |
| **`SetEndpoint`** | 1 | `endpoint_cap` | Bind interrupt to an Endpoint. |
| **`Ack`** | 2 | - | Acknowledge interrupt (EOI). |

## 5. IPC Model

Inter-Process Communication is performed using `sys_send` and `sys_recv`.

*   **`sys_send(cptr, ...)`**:
    *   Sends a message to the Endpoint identified by `cptr`.
    *   Blocks the caller until a receiver is ready.
    *   Transfers data from registers and UTCB to the receiver.
    *   Can transfer capabilities if specified in the UTCB.
*   **`sys_recv(cptr, ...)`**:
    *   Waits for a message on the Endpoint identified by `cptr`.
    *   Blocks the caller until a message arrives.
    *   Receives data into registers and UTCB.
