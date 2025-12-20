# Process Management Design

## 1. Overview

In Glenda's microkernel architecture, the concept of a "Process" is decomposed into orthogonal kernel objects to ensure policy-mechanism separation. The kernel manages **Threads** (execution units), while **Address Spaces** (VSpace) and **Capability Spaces** (CSpace) are resources that threads utilize.

The core kernel object representing a thread of execution is the **TCB (Thread Control Block)**.

## 2. Thread Lifecycle

A thread in Glenda exists in one of several strictly defined states. State transitions are triggered by system calls (IPC), interrupts, or explicit TCB capability invocations.

### 2.1 States

| State | Description |
| :--- | :--- |
| **Inactive** | The TCB is allocated but not yet configured or has been explicitly suspended. It is not eligible for scheduling. |
| **Ready** | The thread is ready to execute and is present in the scheduler's run queue. |
| **Running** | The thread is currently executing on the CPU. |
| **BlockedSend** | The thread is waiting to send an IPC message to a busy receiver. |
| **BlockedRecv** | The thread is waiting to receive an IPC message from an endpoint. |

### 2.2 State Transition Diagram

```text
       +--------+   Configure    +----------+
       | Untyped| -------------> | Inactive | <--------------------+
       +--------+                +----------+                      |
                                   |    ^                          |
                            Resume |    | Suspend                  |
                                   v    |                          |
                              +----------+                         |
                   +--------> |   Ready  | <------------------+    |
                   |          +----------+                    |    |
          Preempt  |               | Schedule                 |    |
          / Yield  |               v                          |    |
                   |          +---------+    IPC Recv         |    |
                   +--------- | Running | ------------------> |    |
                              +---------+                     |    |
                                   |     IPC Send             |    |
                                   +----------------------->  |    |
                                                              |    |
                                                              v    |
                                                       +-------------+
                                                       |   Blocked   |
                                                       | (Send/Recv) |
                                                       +-------------+
```

## 3. Thread Control Block (TCB) Structure

The TCB is a kernel-only structure. It contains the minimal state required to manage execution.

### 3.1 Core Fields

*   **Arch State**: Saved CPU registers (PC, SP, General Purpose Registers).
*   **Priority**: Scheduling priority (0-255).
*   **Time Slice**: Remaining time quantum.
*   **State**: Current lifecycle state (enum).
*   **CSpace Root**: Capability to the root `CNode` of the thread's capability space.
*   **VSpace Root**: Capability to the root `PageTable` of the thread's address space.
*   **UTCB Pointer**: Kernel virtual address of the User Thread Control Block.
*   **Fault Handler**: Capability (Endpoint) to send fault IPCs to.

### 3.2 User Thread Control Block (UTCB)

The UTCB is a page of memory shared between the kernel and the user thread. It is mapped into the user's VSpace.

*   **Purpose**:
    *   **IPC Message Registers**: Storage for IPC payloads that exceed CPU registers.
    *   **TLS**: Thread Local Storage pointer.
    *   **IPC Buffer**: Destination for received capabilities.
*   **Access**:
    *   **User**: Read/Write.
    *   **Kernel**: Read/Write (during IPC and thread setup).

## 4. Lifecycle Operations

### 4.1 Creation

Thread creation is a user-space responsibility (usually performed by a root task or process manager), involving multiple kernel steps:

1.  **Allocation**: A parent thread invokes an `Untyped` capability to **Retype** a portion of memory into a new `TCB` object.
2.  **Address Space Setup**: The parent assigns a VSpace (PageTable cap) to the TCB.
3.  **Capability Space Setup**: The parent assigns a CSpace (CNode cap) to the TCB.
4.  **UTCB Setup**: The parent maps a frame into the new thread's VSpace to serve as the UTCB and registers its address in the TCB.
5.  **Configuration**: The parent invokes the `TCB` capability to set the initial Instruction Pointer (IP), Stack Pointer (SP), and Priority.
6.  **Activation**: The parent invokes `TCB::Resume()`, transitioning the thread from **Inactive** to **Ready**.

### 4.2 Execution & Scheduling

*   **Algorithm**: Preemptive Priority Round-Robin.
*   **Policy**:
    *   Always run the highest priority **Ready** thread.
    *   If multiple threads have the same highest priority, round-robin between them.
*   **Preemption**: Occurs when a higher priority thread becomes Ready (e.g., via interrupt or IPC unblocking) or when the current thread's time slice is exhausted.

### 4.3 Fault Handling

When a thread causes a fault (e.g., Page Fault, Illegal Instruction, Divide by Zero):

1.  The kernel suspends the thread.
2.  The kernel constructs an IPC message describing the fault.
3.  The kernel sends this message to the thread's registered **Fault Handler** (an Endpoint capability).
4.  The thread enters **BlockedSend** state (waiting for the handler to reply).
5.  The Fault Handler (external monitor/debugger) receives the message, decides how to handle it (e.g., kill thread, fix mapping, restart), and replies.
6.  The reply unblocks the thread (or modifies its state).

### 4.4 Termination

Threads do not "exit" in the traditional sense; they simply stop running or are destroyed.

*   **Voluntary Exit**: A thread can call a method on its own TCB cap to suspend itself, or send a message to its manager requesting destruction.
*   **Involuntary Termination**: A manager holding the TCB capability can invoke `TCB::Suspend()` or simply **Revoke** the TCB capability.
*   **Resource Reclamation**:
    *   When a TCB is destroyed (via `Revoke` on the Untyped memory that created it), the kernel ensures it is removed from scheduler queues.
    *   Capabilities held in the thread's CSpace are *not* automatically destroyed unless the CNode itself is destroyed.
    *   The VSpace is *not* automatically destroyed; it is merely detached.

## 5. Capability Interface

The `TCB` kernel object exposes the following methods via `syscall_invoke`:

| Method | Description |
| :--- | :--- |
| `Configure` | Set CSpace root, VSpace root, UTCB address, and Fault Handler. |
| `SetPriority` | Change the scheduling priority. |
| `SetRegisters` | Write to the thread's saved register state (IP, SP, etc.). |
| `GetRegisters` | Read the thread's saved register state. |
| `Resume` | Transition from **Inactive** to **Ready**. |
| `Suspend` | Transition from any state to **Inactive**. |

## 6. Future Work: SMP Support

*   **Affinity**: TCBs will need a CPU affinity field.
*   **Migration**: Mechanism to move TCBs between per-CPU run queues.
*   **IPI**: Inter-Processor Interrupts to trigger rescheduling on other cores.
