# System Call Interface Design

## 1. Overview

The Glenda microkernel uses a unified, capability-based invocation model. Instead of a traditional syscall table indexed by a syscall number (e.g., in `a7`), all kernel services are accessed by invoking a **Capability Pointer (CPtr)**.

## 2. Unified Invocation Interface

There is only one entry point to the kernel via the `ecall` instruction. The kernel determines the operation based on the capability provided in the registers.

### Register Convention (RISC-V)

| Register | Name | Description |
| :--- | :--- | :--- |
| **`a0`** | `cptr` | The Capability Pointer to the target kernel object. |
| **`a7`** | `method` | Method ID |
| **`a1 - a6`** | `args / MRs` | Message Registers (MR0-MR5) for arguments or IPC data. |

## 3. Kernel Object Methods

When a capability is invoked, the `Method ID` determines the operation.

### 3.1 IPC Endpoint
| Method | ID | Description |
| :--- | :--- | :--- |
| **`Send`** | 1 | Send an IPC message to the endpoint. |
| **`Recv`** | 2 | Receive an IPC message from the endpoint. |
| **`Call`** | 3 | Send a message and wait for a reply (RPC). |
| **`Notify`** | 4 | Send a notification (signal) to the endpoint. |

### 3.2 Reply Object
| Method | ID | Description |
| :--- | :--- | :--- |
| **`Reply`** | 1 | Reply to a `Call` operation. |

### 3.3 Thread Control Block (TCB)

| Method | ID | Description |
| :--- | :--- | :--- |
| **`Configure`** | 1 | Set CSpace, VSpace, UTCB and Fault Handler. |
| **`SetPriority`** | 2 | Set the thread's scheduling priority. |
| **`SetRegisters`** | 3 | Set the thread's CPU registers. |
| **`Resume/Suspend`** | 4/5 | Control thread execution state. |

### 3.3 Page Table

| Method | ID | Description |
| :--- | :--- | :--- |
| **`Map`** | 1 | Map a physical page into the page table. |
| **`Unmap`** | 2 | Unmap a page from the page table. |

### 3.4 CNode (Capability Space)

| Method | ID | Description |
| :--- | :--- | :--- |
| **`Mint`** | 1 | Create a new capability with specified rights. |
| **`Copy`** | 2 | Copy an existing capability. |
| **`Delete`** | 3 | Delete a capability. |
| **`Revoke`** | 4 | Revoke a capability and its derivatives. |

### 3.5 Untyped Memory
| Method | ID | Description |
| :--- | :--- | :--- |
| **`Retype`** | 1 | Retype untyped memory into kernel objects. |

### 3.6 IRQ Handler
| Method | ID | Description |
| :--- | :--- | :--- |
| **`SetNotification`** | 1 | Set the notification endpoint for the IRQ. |
| **`Acknowledge`** | 2 | Acknowledge the IRQ. |
| **`ClearNotification`** | 3 | Clear the notification endpoint for the IRQ. |
| **`SetPriority`** | 4 | Set the IRQ priority. |
