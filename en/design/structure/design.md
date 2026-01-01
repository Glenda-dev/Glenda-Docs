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
        Factotum[Factotum (Process/Fault Mgr)]
        Gopher[Gopher (Namespace/VFS)]
        Unicorn[Unicorn (Driver Mgr)]
        Tux[Tux (POSIX Server)]
        App[User Applications]
    end
    subgraph Kernel Space
        Kernel[Glenda Microkernel]
    end

    9Ball --> Kernel
    Factotum --> Kernel
    Gopher --> Kernel
    Unicorn --> Kernel
    App --> Factotum
    App --> Gopher
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
    *   Responsible for starting and bootstrapping other core system services (Factotum, Gopher, Unicorn, etc.).
    *   Allocate resources to the corresponding services.

### 3.2 Factotum (Exception & Task Manager)
*   **Role**: The system's "steward", responsible for process and thread management.
*   **Responsibilities**:
    *   **Exception Handling**: Registered as the Fault Handler for all normal processes. When a page fault or illegal operation occurs, the kernel sends a message to Factotum for handling.
    *   **Process Management**: Responsible for process creation (Spawn), destruction, and lifecycle management.
    *   **Memory Management**: Maintains the address space layout of processes, handles page faults, and implements strategies like Copy-On-Write (COW).

### 3.3 Gopher (Namespace & VFS Server)
*   **Role**: Resource manager, implementing a namespace similar to Plan 9.
*   **Responsibilities**:
    *   **Unified Namespace**: Abstracts all resources (files, devices, network connections, process information) into a file system tree.
    *   **9P2000 Server**: Implements the full 9P2000 protocol over IPC. All resource access is done via standard 9P messages (Tattach, Twalk, Topen, Tread, Twrite).
    *   **Protocol Translation**: Acts as a 9P multiplexer/proxy. It forwards 9P requests to specific resource providers (drivers, file systems) or synthesizes responses for virtual files.

### 3.4 Unicorn (Device Driver Manager)
*   **Role**: Driver manager.
*   **Responsibilities**:
    *   Manage hardware device drivers.
    *   Convert hardware interrupts (IRQ) into IPC messages and send them to driver threads.
    *   Register device files with Gopher.

### 3.5 Tux (POSIX Server)
*   **Role**: POSIX compatibility layer.
*   **Responsibilities**:
    *   Provide standard POSIX system call interfaces for normal applications (forwarded via LibC).
    *   Convert POSIX requests into Glenda's native IPC calls, interacting with Factotum and Gopher.

### 3.6 Rio (Display Manager)
*   **Role**: Graphics display manager.
*   **Responsibilities**:
    *   Manage graphics hardware (GPU/Framebuffer).
    *   Provide windowing system and input event dispatching.

### 3.7 Shell (Command Line Interface)
*   **Role**: The primary user interface for system interaction.
*   **Responsibilities**:
    *   **Command Parsing**: Interprets user commands and scripts.
    *   **Process Control**: Launches applications via Factotum (Spawn).
    *   **File Management**: Interacts with Gopher for file operations.
    *   **Environment**: Manages environment variables and working directories.

## 4. Component Interaction Model

Glenda adopts a Client-Server model. Applications act as Clients, sending requests to Servers via IPC.

*   **System Call Path**: App -> LibC -> IPC -> Tux/Factotum/Gopher -> Kernel
*   **Exception Handling Path**: App (Fault) -> Kernel -> IPC -> Factotum -> (Fix/Kill) -> Kernel -> App
