# Glenda Microkernel Design Document

## 1. Architecture Overview

Glenda is transitioning from a monolithic architecture to an L4-like microkernel architecture. The core philosophy is **Minimality** and the **Separation of Mechanism and Policy**.

The kernel runs in Supervisor Mode (S-Mode) and provides only the most fundamental mechanisms required to construct an operating system. All higher-level abstractions, including device drivers, file systems, and network protocols, are implemented as user-space services.

## 2. Kernel Scope

The microkernel is responsible for the following minimal set of features:

*   **Inter-Process Communication (IPC)**: The fundamental mechanism for data exchange and synchronization between threads.
*   **Address Space Management**: Managing virtual memory mappings and page tables.
*   **Thread Management**: Thread creation, scheduling, and context switching.
*   **Capability System (Cap)**: Fine-grained access control for all kernel objects.
*   **Interrupt Handling**: Converting hardware interrupts into asynchronous notifications sent to specific user-space handlers via the Capability system.

### Components Moved to User Space
*   **Device Drivers**: UART, VirtIO, etc.
*   **File Systems**: VFS, FAT32, etc.
*   **System Services**: Process management, memory servers.

## 3. Key Mechanisms

### 3.1 Capability System (CSpace)
Security and resource management are based on Capabilities. A Capability is a kernel-protected token that refers to a kernel object with specific access rights.

*   **Objects**: TCB (Thread Control Block), Endpoint (IPC Port), Frame (Physical Page), Untyped (Free Memory), IrqHandler.
*   **CSpace**: Each process has a Capability Space (CSpace) storing its capabilities.
*   **Invocation**: System calls are performed by invoking a capability (e.g., `invoke(cptr, args)`).

### 3.2 Inter-Process Communication (IPC)
IPC is the only way for threads to interact.
*   **Synchronous**: IPC is typically synchronous to avoid buffering overhead.
*   **Message Passing**: Short messages are passed via registers; long messages use shared memory buffers.
*   **Notifications**: Asynchronous signals (similar to semaphores) for interrupts.

### 3.3 Exception Handling (Application Interrupts)
The kernel does not handle page faults or other application exceptions internally. Instead, these are treated as **Application Interrupts**.
1.  When a thread faults (e.g., Page Fault), the kernel suspends the thread.
2.  The kernel generates an IPC message describing the fault.
3.  This message is sent to the thread's registered **IRQHandler** (formerly Pager).
4.  The IRQHandler resolves the fault (e.g., allocates a frame) and replies to the kernel.
5.  The kernel resumes the faulted thread.

### 3.4 Root Task
The **Root Task** is the first user-space process started by the kernel.
*   It receives capabilities for all available system resources (all free memory, all IO ports/IRQs).
*   It acts as the initial Resource Manager and IRQHandler.
*   It is responsible for bootstrapping the rest of the user-space environment (starting drivers, FS server, shell).

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

## 5. System Call Interface (Draft)

*   `sys_yield()`: Relinquish CPU.
*   `sys_send(dest_cptr, msg)`: Send IPC.
*   `sys_recv(src_cptr, buf)`: Receive IPC.
*   `sys_call(cptr, msg)`: Atomic Send + Recv (RPC style).
*   `sys_reply(cptr, msg)`: Reply to a call (non-blocking).
