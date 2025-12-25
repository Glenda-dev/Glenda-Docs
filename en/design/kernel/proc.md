# Process Management Design

## 1. Overview

In Glenda's microkernel architecture, the concept of a "Process" is decomposed into orthogonal kernel objects to ensure policy-mechanism separation. The kernel manages **Threads** (execution units), while **Address Spaces** (VSpace) and **Capability Spaces** (CSpace) are resources that threads utilize.

The core kernel object representing a thread of execution is the **TCB (Thread Control Block)**. This abstraction is used for both user-space processes and internal kernel tasks (e.g., Idle thread, Interrupt handlers).

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
*   **Privileged**: Boolean indicating if the thread is a **Kernel Thread**.
    *   If `true`, the thread runs in **Supervisor Mode (S-Mode)** and uses the kernel's address space.
    *   If `false`, the thread runs in **User Mode (U-Mode)** and uses its assigned `VSpace`.
*   **CSpace Root**: Capability to the root `CNode` of the thread's capability space.
*   **VSpace Root**: Capability to the root `PageTable` of the thread's address space.
*   **UTCB Frame**: Capability to the physical frame used for the UTCB.
*   **UTCB Pointer**: Virtual address of the User Thread Control Block in the thread's address space.
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
4. **UTCB Setup**: The parent allocates a frame to serve as the UTCB and maps it into the new thread's VSpace.
5. **Configuration**: The parent invokes the `TCB` capability to bind the CSpace, VSpace, UTCB frame, and UTCB virtual address. It also sets the initial Instruction Pointer (IP), Stack Pointer (SP), and Priority.
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

### 4.5 Kernel Threads & Context Switching

Glenda treats kernel tasks as special TCBs with the `privileged` flag set to `true`.

#### 4.5.1 Kernel TCB Characteristics
*   **Privilege Level**: Always executes in S-Mode.
*   **Address Space**: Typically shares the kernel's global page table.
*   **Stack**: Uses a dedicated kernel stack.
*   **CSpace**: May have a restricted or null CSpace as it operates with kernel authority.

#### 4.5.2 Switching Logic
The scheduler performs context switches based on the `privileged` flag of the target TCB:

1.  **User -> User**:
    *   Save current U-Mode registers to current TCB.
    *   Switch `satp` to target VSpace.
    *   Restore target U-Mode registers.
    *   `sret` to U-Mode.
2.  **User -> Kernel**:
    *   Trap occurs (System Call/Interrupt).
    *   Save U-Mode state to current TCB.
    *   Switch to target Kernel TCB stack.
    *   Restore Kernel TCB registers.
    *   Continue execution in S-Mode.
3.  **Kernel -> User**:
    *   Save Kernel TCB state.
    *   Switch `satp` to target VSpace.
    *   Restore target U-Mode state.
    *   `sret` to U-Mode.
4.  **Kernel -> Kernel**:
    *   Directly save/restore S-Mode registers (callee-saved) and switch stacks. No `satp` switch is required if they share the kernel address space.

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
