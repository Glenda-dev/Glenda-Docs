# Glenda Microkernel Design Document

## 1. Architecture Overview

Glenda is an L4-like microkernel architecture. The core philosophy is **Minimality** and the **Separation of Mechanism and Policy**.

The kernel runs in Supervisor Mode (S-Mode) and provides only the most fundamental mechanisms required to construct an operating system. All higher-level abstractions, including device drivers, file systems, and network protocols, are implemented as user-space services.

## 2. Kernel Scope

The microkernel is responsible for the following minimal set of features:

*   **Inter-Process Communication (IPC)**: The fundamental mechanism for data exchange and synchronization between threads.
*   **Address Space Management**: Managing virtual memory mappings and page tables.
*   **Thread Management**: Thread creation, scheduling, and context switching.
*   **Capability System (Cap)**: Fine-grained access control for all kernel objects.
*   **Interrupt Handling**: Converting hardware interrupts into IPC messages sent to specific user-space handlers via the Capability system.

### Components Moved to User Space
*   **Device Drivers**: UART, VirtIO, etc.
*   **File Systems**: VFS, FAT32, etc.
*   **System Services**: Process management, memory servers.

## 3. Key Mechanisms

### 3.1 Capability System (CSpace)
Security and resource management are based on Capabilities. A Capability is a kernel-protected token that refers to a kernel object with specific access rights.

*   **Lifecycle Management**: Kernel objects are managed via **Reference Counting**. The `Capability` structure implements `Clone` (increment ref) and `Drop` (decrement ref). When the count reaches zero, the underlying physical memory is reclaimed.
*   **Untyped Memory**: All physical memory is initially represented as `Untyped` capabilities. Kernel objects (TCB, CNode, etc.) are created by "retyping" `Untyped` memory. This unifies memory management with the capability system.
*   **Badge**: A 64-bit immutable identifier attached to a capability during a `Mint` operation.
    *   **Identity**: Servers use badges to identify clients.
    *   **Immutability**: Once a badge is set, it cannot be modified or removed by the holder.
    *   **Notification**: For notification objects, badges act as bitmasks that are ORed together upon signaling.
*   **Objects**: TCB (Thread Control Block), Endpoint (IPC Port), Frame (Physical Page), Untyped (Free Memory), IrqHandler, PageTable, CNode.
*   **CSpace**: Each process has a Capability Space (CSpace) storing its capabilities.
*   **Invocation**: System calls are performed by invoking a capability (e.g., `sys_invoke(cptr, args)`).

### 3.2 Inter-Process Communication (IPC)
IPC is the only way for threads to interact.
*   **Synchronous**: IPC is typically synchronous to avoid buffering overhead.
*   **Message Passing**: Short messages are passed via registers; extended messages and capabilities are passed via the **UTCB** (User Thread Control Block).
*   **Badge Passing**: During IPC, the kernel extracts the badge from the sender's capability and injects it into the receiver's context (e.g., register `t0`). This provides a secure, kernel-verified identity.
*   **Endpoints**: IPC is directed towards Endpoint objects, not threads directly.

### 3.3 Exception Handling (Application Interrupts)
The kernel does not handle page faults or other application exceptions internally. Instead, these are treated as **Application Interrupts**.
1.  When a thread faults (e.g., Page Fault), the kernel suspends the thread.
2.  The kernel saves fault details to the thread's UTCB.
3.  The kernel generates an IPC message describing the fault.
4.  This message is sent to the thread's registered **Exception Handler** (an Endpoint capability stored in the TCB).
5.  The Handler resolves the fault (e.g., allocates a frame) and replies to the kernel.
6.  The kernel resumes the faulted thread.

### 3.4 Root Task
The **Root Task** is the first user-space process started by the kernel.
*   It receives capabilities for all available system resources (all free memory, all IO ports/IRQs).
*   **CSpace Layout**: The Root Task has a fixed initial CSpace layout:
    *   Slot 0: Root CNode Capability
    *   Slot 1: Root VSpace (PageTable) Capability
    *   Slot 2: Root TCB Capability
    *   Slot 3: Root UTCB Frame Capability
    *   Slot 4+: Untyped Memory Capabilities
*   It acts as the initial Resource Manager and IRQHandler.
*   It is responsible for bootstrapping the rest of the user-space environment (starting drivers, FS server, shell).

### 3.5 Kernel Abstractions as TCBs
The kernel itself uses the TCB abstraction for internal management:
*   **Idle Thread**: A lowest-priority TCB per CPU that runs when no other threads are ready.
*   **Kernel Threads**: While the kernel is mostly event-driven, it can support kernel-mode threads for specific background tasks if necessary, though the primary design avoids them in favor of user-space handlers.

## 4. Boot Process

1.  **Bootloader**:
    *   **OpenSBI (RISC-V)**: Jumps directly to Kernel. Kernel uses embedded payload.
    *   **GRUB (x86/UEFI)**: Loads Kernel and `payload.bin`. Passes Multiboot info.
2.  **Kernel Init**:
    *   Initialize CPU, Trap Vector, Memory Allocator.
    *   **Payload Discovery**:
        *   Check Multiboot info for modules.
        *   Fallback to embedded `_payload_start`.
    *   Create the Root Task.
    *   Populate Root Task's CSpace with all resources.
3.  **Switch to User Mode**: Kernel jumps to Root Task entry point.
4.  **User Space Init**:
    *   Root Task initializes its allocator.
    *   Root Task spawns Driver processes (UART, Timer).
    *   Root Task spawns File System Server.
    *   Root Task spawns Shell/Init process.

## 5. System Call Interface

*   **`sys_invoke(cptr, ...)`**: Invoke a method on a kernel object (TCB, PageTable, etc.).
