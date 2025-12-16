# Inter-Process Communication (IPC) Design

## 1. Overview

In Glenda's microkernel architecture, Inter-Process Communication (IPC) is the fundamental mechanism for coordination and data exchange. Unlike monolithic kernels where components call functions directly, microkernel components (drivers, filesystems, apps) run in isolated address spaces and must use IPC to interact.

## 2. Design Principles

*   **Synchronous by Default**: To minimize buffering overhead and complexity in the kernel, standard IPC is synchronous. The sender blocks until the receiver is ready, and the receiver blocks until a message arrives. Handover is direct.
*   **Zero-Copy (where possible)**: Short messages are passed purely in registers. Longer messages use shared memory buffers or direct copy between address spaces if necessary.
*   **Performance Critical**: IPC is the "hot path" of the kernel.

## 3. Message Structure

An IPC message consists of:

1.  **Message Tag (MR0)**: Describes the message protocol, length, and flags.
2.  **Short Message (Registers)**: Up to 8 machine words passed directly in CPU registers (e.g., `a0`-`a7` on RISC-V). This is sufficient for most system calls and simple protocols.
3.  **Long Message (Optional)**: For larger data transfers, a shared memory buffer (IPC Buffer) defined in the thread's UTCB (User Thread Control Block) is used.
4.  **Capabilities (Optional)**: A message can transfer a Capability from the sender's CSpace to the receiver's CSpace (Cap Delegation).

## 4. IPC Operations

The system call interface provides the following primitives:

*   **`Send(dest_cptr, msg)`**:
    *   Blocking send.
    *   Blocks until `dest` is ready to receive.
*   **`Recv(src_cptr, buf)`**:
    *   Blocking receive.
    *   Blocks until a message arrives from `src` (or any open Endpoint).
*   **`Call(dest_cptr, msg)`**:
    *   Atomic `Send` + `Recv`.
    *   Used for RPC (Remote Procedure Call).
    *   The kernel guarantees that the reply comes from the callee.
*   **`Reply(msg)`**:
    *   Non-blocking send to the thread that was just `Recv`'d from.
    *   Used by servers to return results without blocking.
*   **`ReplyRecv(msg)`**:
    *   Atomic `Reply` + `Recv`.
    *   The standard loop for server threads: "Reply to previous client, then wait for next request."

## 5. Endpoints

IPC is not directed at "Thread IDs" but at **Endpoints**.
*   An **Endpoint** is a kernel object that acts as a mailbox or rendezvous point.
*   Threads holding a `Send` capability to an Endpoint can send messages.
*   Threads holding a `Recv` capability to an Endpoint can listen for messages.
*   This allows multiple clients to talk to a single server without knowing the server's internal thread ID.

## 6. Notifications (Asynchronous IPC)

For interrupts and simple signals, full synchronous IPC is too heavy.
*   **Notification Objects**: Similar to semaphores or event bits.
*   **Signal**: Non-blocking. Sets a bit in the receiver's notification word.
*   **Wait**: Blocking. Wakes up when a bit is set.
*   Hardware interrupts are delivered as Notifications from the kernel to the driver thread.
