# System Call Interface

## 1. Overview

Glenda uses a unified, capability-based invocation model. All kernel services are accessed by invoking a **Capability Pointer (CPtr)** via the `ecall` instruction.

### Register Convention (RISC-V)

| Register | Name | Description |
| :--- | :--- | :--- |
| **`a0`** | `cptr` | The Capability Pointer to the invoked object. |
| **`a7`** | `method` | Method ID (specific to object type). |
| **`a1 - a6`** | `args` | Arguments (MR0-MR5 in UTCB, but passed via registers for fastpath). |

> Note: Arguments are typically marshaled into the UTCB by the user library (`libglenda`), but some fastpath calls may use registers directly.

## 2. Object Methods

### 2.1 IPC Endpoint (`Endpoint`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`Send`** | 1 | - | Send an IPC message. Blocks if no receiver. |
| **`Recv`** | 2 | - | Wait for an IPC message. |
| **`Call`** | 3 | - | Send and wait for reply. |
| **`Notify`**| 4 | - | Send a notification (signal). |

### 2.2 Thread Control Block (`TCB`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`Configure`** | 1 | `cspace, vspace, utcb, tf, kstack` | Set capability spaces and buffers. |
| **`SetPriority`**| 2 | `prio` | Set scheduling priority (0-255). |
| **`SetEntry`**   | 3 | `pc, sp, tp` | Set thread entry point and stack pointer. |
| **`SetFaultHandler`**| 4 | `ep_cptr` | Set exception handler endpoint. |
| **`SetAffinity`**| 5 | `cpu_id` | Set CPU affinity. |
| **`SetRegisters`**| 6 | `regs...` | Write generic registers. |
| **`Resume`** | 7 | - | Resume thread execution. |
| **`Suspend`** | 8 | - | Suspend thread execution. |

### 2.3 Capability Node (`CNode`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`Mint`** | 1 | `src, dest, badge, rights` | Create a new cap with optional badge/rights. |
| **`Copy`** | 2 | `src, dest, rights` | Copy a cap to another slot. |
| **`Delete`** | 3 | `slot` | Delete a cap from a slot. |
| **`Revoke`** | 4 | `slot` | Recursively delete children of a cap. |
| **`Debug`** | 5 | - | Dump CNode info to kernel log. |

### 2.4 Virtual Space (`VSpace`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`Map`** | 1 | `frame_cap, vaddr, flags` | Map a frame into virtual address space. |
| **`Unmap`** | 2 | `vaddr, size` | Unmap a range of virtual memory. |
| **`MapTable`**| 3 | `table_cap, vaddr, level` | Map a lower-level page table. |
| **`Setup`** | 4 | - | Initialize VSpace root. |
| **`Debug`** | 5 | - | Dump PageTable info to kernel log. |

### 2.5 Page Table (`PageTable`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`MapTable`**| 1 | `table_cap, vaddr, level` | Link a sub-page table. |

### 2.6 Untyped Memory (`Untyped`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`Retype`** | 1 | `type, flags, cnode, slot` | Create objects from untyped memory. |

### 2.7 Interrupt Handler (`IrqHandler`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`SetNotify`**| 1 | `ep_cptr` | Bind IRQ to a notification endpoint. |
| **`Ack`** | 2 | - | Acknowledge interrupt (EOI). |
| **`Clear`** | 3 | - | Unbind notification. |
| **`SetPriority`**| 4 | `prio` | Set IRQ hardware priority. |

### 2.8 Kernel Resource (`Kernel`)

| Method | ID | Arguments | Description |
| :--- | :-- | :--- | :--- |
| **`PutStr`** | 1 | - | Write string to debug console. |
| **`GetChar`** | 2 | - | Read char from debug console. |
| **`GetStr`** | 3 | - | Read string from debug console. |
| **`Shell`** | 4 | - | Enter kernel debug shell. |
