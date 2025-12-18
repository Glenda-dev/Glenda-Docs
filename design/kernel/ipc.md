# Inter-Process Communication (IPC) Design

## 1. Overview

In Glenda's microkernel architecture, Inter-Process Communication (IPC) is the fundamental mechanism for coordination and data exchange. Unlike monolithic kernels where components call functions directly, microkernel components (drivers, filesystems, apps) run in isolated address spaces and must use IPC to interact.

## 2. Design Principles

*   **Synchronous by Default**: To minimize buffering overhead and complexity in the kernel, standard IPC is synchronous. The sender blocks until the receiver is ready, and the receiver blocks until a message arrives. Handover is direct.
*   **Zero-Copy (where possible)**: Short messages are passed purely in registers. Longer messages use shared memory buffers or direct copy between address spaces if necessary.
*   **Performance Critical**: IPC is the "hot path" of the kernel.

## 3. Message Structure

An IPC message consists of:

1.  **Message Tag (MR0)**: Describes the message protocol, length, and flags. Passed in a register.
2.  **Short Message (Registers)**: Up to 8 machine words passed directly in CPU registers (e.g., `a0`-`a7` on RISC-V). This is sufficient for most system calls and simple protocols.
3.  **Extended Message (UTCB)**: If the message exceeds the register count, the remaining words are stored in the thread's UTCB (Message Registers MR8+). The kernel copies these from the sender's UTCB to the receiver's UTCB.
4.  **Capabilities (Optional)**: A message can transfer a Capability from the sender's CSpace to the receiver's CSpace (Cap Delegation).
    *   **Sender**: Writes a **Cap Transfer Descriptor** (CTD) into its UTCB. The CTD contains the CPTR of the cap to send.
    *   **Receiver**: Writes a **Receive Window Descriptor** (RWD) into its UTCB. The RWD specifies the CNode and index where the received cap should be placed.
    *   **Kernel**: Validates the transfer (checking for `Grant` rights) and performs the copy/move operation during the IPC handshake.

## 4. IPC Operations

The system call interface provides the following primitives:

*   **`Send(dest_cptr, msg)`**:
    *   Blocking send.
    *   Blocks until `dest` is ready to receive.
*   **`Recv(src_cptr, buf)`**:
    *   Blocking receive.
    *   Blocks until a message arrives from `src` (or any open Endpoint).

## 5. Endpoints

IPC is not directed at "Thread IDs" but at **Endpoints**.
*   An **Endpoint** is a kernel object that acts as a mailbox or rendezvous point.
*   Threads holding a `Send` capability to an Endpoint can send messages.
*   Threads holding a `Recv` capability to an Endpoint can listen for messages.
*   This allows multiple clients to talk to a single server without knowing the server's internal thread ID.
