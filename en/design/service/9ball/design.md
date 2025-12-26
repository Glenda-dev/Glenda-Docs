# 9Ball: Root Task Design

## 1. Overview
9Ball is the Root Task (Initial Process) of the Glenda operating system. It is the first user-space program loaded by the kernel. It serves as the system's "Init" process, responsible for bootstrapping the user-space environment and managing global resources.

## 2. Responsibilities
1.  **Resource Management**: It receives all system resources (Memory, IRQs) from the kernel and acts as the primary allocator.
2.  **Service Bootstrapping**: It spawns essential system services (Unicorn, Gopher, Tux) according to the roadmap.
3.  **Capability Distribution**: It enforces the principle of least privilege by distributing only necessary capabilities to each service.
4.  **Name Service**: It acts as a simple name server, allowing services to find each other's IPC endpoints.

## 3. Initial State
Upon startup, the Kernel grants 9Ball capabilities to all available hardware resources. 9Ball's CSpace initially contains:
*   **Untyped Memory**: Capabilities covering all free physical RAM.
*   **Device Memory**: Capabilities covering MMIO regions (derived from DTB).
*   **IRQ Handlers**: Capabilities for all hardware interrupts.
*   **BootInfo**: Information about the hardware layout and memory map.

## 4. Architecture & Modules

### 4.1 Memory Manager (MemMgr)
*   Manages the `Untyped` capabilities.
*   Implements a physical memory allocator (e.g., Buddy System or Slab) on top of `Untyped::Retype`.
*   Other services request memory (Frames, TCBs, CNodes) from MemMgr via IPC.

### 4.2 Process Manager (ProcMgr)
*   Responsible for parsing ELF files and loading other services.
*   Creates TCBs, VSpaces, and CSpaces for new processes.
*   Handles the "spawn" logic: `Retype TCB` -> `Map Memory` -> `Set Priority` -> `Resume`.

### 4.3 Service Manager (SvcMgr)
*   Orchestrates the boot sequence.
*   Maintains a registry of running services and their IPC endpoints.

### 4.4 Hardware Discovery (DTB Parser)
*   **Input**: Receives the Device Tree Blob (DTB) from the Kernel via BootInfo.
*   **Parsing**: Scans the DTB to identify:
    *   **Memory Regions**: To initialize the Memory Manager.
    *   **MMIO Regions**: To create `Device Frame` capabilities for peripherals.
    *   **IRQ Mappings**: To correlate Interrupt Controllers with devices (though 9Ball delegates all IRQs to Unicorn).
*   **Handoff**: Passes the raw DTB to Unicorn so it can perform driver matching.

## 5. Permission Distribution (Roadmap Implementation)

According to the system roadmap, 9Ball distributes permissions to the core services as follows:

### 5.1 Unicorn (Device Driver Manager)
Unicorn is responsible for managing hardware drivers (UART, VirtIO, etc.). It acts as the Hardware Abstraction Layer.

*   **Capabilities Granted**:
    *   **DTB Access**: Read-only access to the Device Tree Blob.
    *   **IRQ Handlers**: 9Ball delegates **all** `IrqHandler` capabilities to Unicorn. Unicorn is responsible for further delegating them to specific driver processes or handling them internally.
    *   **MMIO Regions**: Access to device memory regions (e.g., PLIC, UART base, VirtIO MMIO). 9Ball mints `Device Frame` capabilities for these regions based on its DTB scan.
    *   **DMA Memory**: A pool of contiguous `Untyped` memory (or pre-allocated Frames) for DMA buffers.
*   **IPC Connections**:
    *   Exposes endpoints for other services to discover devices (e.g., "Get Serial Port", "Get Block Device").

### 5.2 Gopher (VFS Server)
Gopher implements the Virtual File System and manages file storage.

*   **Capabilities Granted**:
    *   **Memory Quota**: A fixed amount of RAM for file system caches (dentry, inode, page cache).
    *   **No Hardware Access**: Gopher does **not** receive IRQ or MMIO capabilities directly.
*   **IPC Connections**:
    *   **To Unicorn**: Gopher holds a connection to Unicorn to access Block Device drivers (e.g., VirtIO-Blk).
    *   **From Clients**: Gopher exposes an endpoint for file system operations (`open`, `read`, `write`).

### 5.3 Tux (POSIX Server)
Tux provides a POSIX-compatible environment (Process IDs, Signals, Pipes).

*   **Capabilities Granted**:
    *   **Process Creation**: Tux needs the ability to spawn user applications.
        *   *Mechanism*: Tux requests 9Ball to create a new process container (TCB + VSpace). 9Ball returns the control capabilities to Tux.
    *   **Memory Quota**: A dynamic quota of `Untyped` memory to back user processes.
*   **IPC Connections**:
    *   **To Gopher**: For all file-related syscalls.
    *   **To Unicorn**: For TTY/Console access.
    *   **From User Apps**: Tux acts as the syscall server for POSIX applications.

### 5.4 LINE (Linux Compatible Layer)
LINE allows running unmodified Linux binaries by emulating the Linux syscall ABI.

*   **Capabilities Granted**:
    *   Similar to Tux, it acts as a personality server.
    *   It needs broad IPC access to emulate Linux syscalls by translating them to Glenda service calls (e.g., translating a Linux `open` to a Gopher IPC call).

## 6. Boot Sequence

1.  **9Ball Init**:
    *   Initialize MemMgr (scan `Untyped` caps).
    *   Initialize ProcMgr.
2.  **Start Unicorn**:
    *   Load `unicorn` ELF.
    *   Transfer IRQ and MMIO capabilities.
    *   Wait for Unicorn to initialize basic drivers (Serial).
3.  **Start Gopher**:
    *   Load `gopher` ELF.
    *   Establish IPC channel between Gopher and Unicorn.
4.  **Start Tux**:
    *   Load `tux` ELF.
    *   Establish IPC channels to Gopher and Unicorn.
5.  **Start Shell**:
    *   Tux spawns the initial shell (`/bin/sh`).
