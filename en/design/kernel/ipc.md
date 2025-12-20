# Inter-Process Communication (IPC) Design

## 1. Overview

In Glenda's microkernel architecture, Inter-Process Communication (IPC) is the fundamental mechanism for coordination and data exchange. Unlike monolithic kernels where components call functions directly, microkernel components (drivers, filesystems, apps) run in isolated address spaces and must use IPC to interact.

## 2. Design Principles

*   **Synchronous (Rendezvous)**: Standard IPC is synchronous. The sender blocks until a receiver is ready, and vice versa. Data is copied directly between threads during the handshake.
*   **Asynchronous Notifications**: Used for hardware interrupts and simple signaling. These are non-blocking and only carry a "Badge" (identity/event code) without a data payload.
*   **Zero-Copy for Short Messages**: The most frequent messages are passed entirely through CPU registers.
*   **Capability-Based**: IPC is performed by invoking capabilities to **Endpoints**.

## 3. Message Structure

An IPC message is defined by the **UTCB (User Thread Control Block)** layout:

1.  **Message Tag (`msg_tag`)**: A single word describing the message protocol, label, and the number of words in the message.
    *   **Length (Bits 0-3)**: Number of message registers (`mrs_regs`) used (0 to 7).
    *   **Flags (Bits 4-15)**:
        *   Bit 4: `HasCap` - Indicates a capability is being transferred.
        *   Bit 5: `HasBuffer` - Indicates the IPC buffer is used for large payloads.
    *   **Label (Bits 16-63)**: A protocol-specific identifier. For example, `0xFFFF` is reserved for kernel-generated Fault messages.

2.  **Message Registers (`mrs_regs`)**: 7 machine words (MR1-MR7) passed directly in CPU registers (e.g., `a1`-`a7` on RISC-V) during the trap.
3.  **Capability Transfer Fields**:
    *   **`cap_transfer`**: The CPTR of the capability the sender wishes to transfer.
    *   **`recv_window`**: The CPTR (CNode + Index) where the receiver wishes to store an incoming capability.
4.  **IPC Buffer**: A page-aligned buffer used for larger payloads. If `ipc_buffer_size` in the UTCB is non-zero, the kernel performs a `memcpy` from the sender's buffer to the receiver's buffer.
5.  **Badge**: A machine word (passed in `t0`) that identifies the sender's capability or the interrupt source.

## 4. IPC Operations

### 4.1 Synchronous Send/Receive
*   **`sys_send(dest_cptr)`**: Blocks the caller until a receiver is waiting on the target Endpoint.
*   **`sys_recv(src_cptr)`**: Blocks the caller until a sender arrives at the target Endpoint or a pending notification is available.
*   **`sys_call(dest_cptr)`**: (Planned) A combined Send-then-Receive operation that provides an implicit Reply capability for the server to respond.

### 4.2 Asynchronous Notify
*   **`notify(endpoint, badge)`**: A kernel-internal or user-space operation that delivers a badge to an Endpoint without blocking. If no receiver is waiting, the badge is queued in the Endpoint's `pending_notifs` list.

## 5. Endpoints

Endpoints are the rendezvous points for IPC.

*   **`send_queue`**: List of threads blocked while trying to send to this endpoint.
*   **`recv_queue`**: List of threads blocked while waiting for a message on this endpoint.
*   **`pending_notifs`**: A FIFO of badges from asynchronous notifications (e.g., IRQs) that haven't been received yet.

## 6. Capability Delegation

A message can transfer a Capability from the sender's CSpace to the receiver's CSpace. This is essential for resource management and security delegation.

*   **Sender**:
    *   Sets the `HasCap` flag in the `MsgTag`.
    *   Places the CPTR of the capability to be sent in `utcb.cap_transfer`.
    *   Must have `Grant` rights on the capability.
*   **Receiver**:
    *   Specifies a destination slot in its CSpace via `utcb.recv_window`.
*   **Kernel**:
    *   During the Rendezvous (in `copy_msg`), the kernel looks up the capability in the sender's CSpace.
    *   It verifies the `Grant` right.
    *   It inserts the capability into the receiver's CSpace at the specified `recv_window`.
    *   The capability is effectively "moved" or "copied" depending on the specific object type and derivation rules.
