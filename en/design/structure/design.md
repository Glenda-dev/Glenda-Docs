# Glenda System Architecture Design

Glenda is a capability-based microkernel operating system. Its design philosophy follows the minimization principle of microkernels: the kernel provides only the most basic mechanisms, while moving policies and most system services to user space.

## 1. Overall Architecture

Glenda's system architecture is divided into two layers:
1.  **Kernel Space**: Glenda Microkernel
2.  **User Space**: System services and applications

\`\`\`mermaid
graph TD
    subgraph User Space
        9Ball[9Ball (Root Task)]
        Warren[Warren (Process/Fault Mgr)]
        9P[9P (Namespace Server)]
        Gopher[Gopher (Network Stack)]
        Fossil[Fossil (File System)]
        Unicorn[Unicorn (Driver Mgr)]
        Chimera[Chimera (Virtualization)]
        Tux[Tux (L4Linux Server)]
        App[User Applications]
    end
    subgraph Kernel Space
        Kernel[Glenda Microkernel]
    end

    9Ball --> Kernel
    Warren --> Kernel
    9P --> Kernel
    Gopher --> Kernel
    Fossil --> Kernel
    Unicorn --> Kernel
    Chimera --> Kernel
    Tux --> Kernel
    App --> Warren
    App --> 9P
    App --> Gopher
    App --> Fossil
    App --> Chimera
    App --> Tux
\`\`\`

## 2. Kernel Layer (Microkernel)

The kernel is primarily responsible for managing the most basic hardware resources and providing controlled access mechanisms.

### Core Objects
*   **TCB (Thread Control Block)**: Thread execution context.
*   **Endpoint**: IPC communication endpoint, used for message passing between threads.
*   **CNode (Capability Node)**: Container for storing Capabilities (similar to a file descriptor table).
*   **PageTable / Frame**: Memory management objects.
*   **Untyped**: Untyped physical memory, used to derive other objects.
*   **Interrupt**: Interrupt management object.

### Core Mechanisms
*   **Capability-based Security**: All resource access must be done through Capabilities, implementing fine-grained permission control.
*   **IPC (Inter-Process Communication)**: Synchronous and asynchronous message passing mechanisms.
*   **Preemptive Scheduling**: Priority-based preemptive scheduling.

## 3. User Space Service Components

Glenda's functionality is primarily provided by a set of cooperating user-space services.

### 3.1 9Ball (Root Task / System Bootstrapper)
*   **Role**: The first user-space process of the system (PID 1).
*   **Responsibilities**:
    *   Take over all remaining system resources handed over by the kernel (Untyped Memory, IO, IRQ Caps).
    *   Responsible for starting and bootstrapping other core system services (Warren, Gopher, Unicorn, etc.).
    *   Allocate resources to the corresponding services.

### 3.2 Warren (Exception & Task Manager)
*   **Role**: The system's "steward", responsible for process and thread management.
*   **Responsibilities**:
    *   **Exception Handling**: Registered as the Fault Handler for all normal processes. When a page fault or illegal operation occurs, the kernel sends a message to Warren for handling.
    *   **Process Management**: Responsible for process creation (Spawn), destruction, and lifecycle management.
    *   **Memory Management**: Maintains the address space layout of processes, handles page faults, and implements strategies like Copy-On-Write (COW).

### 3.3 9P (Namespace Server)
*   **Role**: Namespace Manager for the system.
*   **Responsibilities**:
    *   **Unified Namespace**: Manages the global namespace tree, mounting resources from other services (Fossil, Gopher, Unicorn) into a unified hierarchy.
    *   **Mount Management**: Handles mount/unmount operations.
    *   **9P2000 Router**: Routes 9P requests to the appropriate service.

### 3.4 Gopher (Network Stack)
*   **Role**: Network stack provider.
*   **Responsibilities**:
    *   **Network Management**: Manages network interfaces and routing.
    *   **TCP/IP Stack**: Implements the TCP/IP protocol stack.
    *   **9P Interface**: Exposes network stack management interface via 9P2000.
    *   **Dedicated IPC**: Uses high-performance shared memory IPC for data path.

### 3.5 Fossil (File System Server)
*   **Role**: Persistent file system server.
*   **Responsibilities**:
    *   **Storage Management**: Manages disk storage.
    *   **File System**: Implements the file system logic.
    *   **9P Interface**: Exposes file system structure via 9P2000.
    *   **Dedicated IPC**: Uses specialized IPC protocols for bulk data transfer.

### 3.6 Unicorn (Device Driver Manager)
*   **Role**: Driver manager.
*   **Responsibilities**:
    *   Manage hardware device drivers.
    *   Convert hardware interrupts (IRQ) into IPC messages.
    *   **9P Interface**: Exposes device files via 9P2000.

### 3.7 Tux (Linux Compatibility Layer)
*   **Role**: Linux Paravirtualization Compatibility Layer.
*   **Responsibilities**:
    *   **Linux Guest**: Runs as a lightweight paravirtualized Linux guest on top of Chimera.
    *   **ABI Compatibility**: Provides Linux ABI compatibility for unmodified Linux binaries.
    *   **9P Interface**: Exposes process info/status via 9P2000.

### 3.8 Rio (Display Manager)
*   **Role**: Graphics display manager.
*   **Responsibilities**:
    *   Manage graphics hardware (GPU/Framebuffer).
    *   Provide windowing system and input event dispatching.
    *   **9P Interface**: Exposes window management via 9P2000.

### 3.9 Shell (Command Line Interface)
*   **Role**: The primary user interface for system interaction.
*   **Responsibilities**:
    *   **Command Parsing**: Interprets user commands and scripts.
    *   **Process Control**: Launches applications via Warren (Spawn).
    *   **File Management**: Interacts with the Namespace (9P) for file operations.

### 3.10 APE (ANSI/POSIX Environment)
*   **Role**: System Library Support.
*   **Responsibilities**:
    *   **Musl Backend**: Serves as the underlying library for `musl`, implementing the platform adapter.
    *   **Direct Interaction**: Interacts directly with system services (Gopher, Fossil, Tux) via their specific dedicated IPC protocols, bypassing the general 9P namespace for performance where possible.
    *   **Protocol Adapter**: Adopts various protocols (e.g., Block I/O, Network Streaming, Framebuffer) to standard POSIX calls.
    *   **Environment**: Manages environment variables and working directories.

### 3.11 Chimera (Virtualization Server)
*   **Role**: Virtual Machine / Container Monitor (VMM).
*   **Responsibilities**:
    *   **Virtualization**: providing hardware virtualization support (using hardware extensions like RISC-V H-Extension or software emulation).
    *   **Resource Isolation**: Manages isolated environments for guest operating systems or containers.
    *   **Tux Host**: Serves as the hypervisor/host for the **Tux** Linux compatibility layer.
    *   **9P Interface**: Exposes VM management via 9P2000.

## 4. Component Interaction Model

Glenda adopts a hybrid Client-Server model.

*   **Data Plane**: Components communicate using **Dedicated IPC Protocols** (e.g., shared memory rings, direct capability invocations) for high-throughput and low-latency operations.
*   **Control/Management Plane**: All components expose a **9P2000** interface. This provides a uniform way to inspect, configure, and manage every part of the system as a file. The **9P Server** aggregates these into a single namespace.

*   **System Call Path**: App -> LibC -> APE -> IPC (Dedicated Protocol) -> Gopher/Fossil/Chimera -> Kernel
*   **Exception Handling Path**: App (Fault) -> Kernel -> IPC -> Warren -> (Fix/Kill) -> Kernel -> App
