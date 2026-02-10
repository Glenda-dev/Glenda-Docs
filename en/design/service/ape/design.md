# APE (ANSI/POSIX Environment) Design Document

## 1. Introduction
APE is the **POSIX Compatibility Shim** for Glenda. Unlike traditional monolithic POSIX servers, APE is primarily implemented as a **Client-Side Library** (part of `libglenda` or `musl-glenda`). It intercepts standard POSIX system calls and directs them to the appropriate Glenda native services.

## 2. Architecture

APE acts as a "Smart Proxy" running within the application's process space.

### 2.1 Library Layer
*   **Location**: `lib/libglenda-rs/src/ape/` or integrated into `musl-glenda` syscall backend.
*   **Function**: Translates C-ABI syscalls/libc functions into Glenda IPC messages.

### 2.2 Service Routing
APE maintains a routing table in the process's user-space memory, mapping file descriptors (FDs) to Glenda Capabilities (Endpoints).

| FD Range | Type | Backend Service | Protocol |
| :--- | :--- | :--- | :--- |
| `0, 1, 2` | Stdio | **Warren** / **Rio** | Ring Buffer / serialized IO |
| `3...N` | Files | **Fossil** | FS Protocol + SHM |
| `M...P` | Sockets | **Gopher** | Dedicated Socket IPC |
| `X...Y` | Devices | **Unicorn** | Device Specific IPC |

## 3. Key Components

### 3.1 VFS Shim (Virtual File System)
Glenda apps don't speak to a central VFS kernel object. APE maintains a lightweight VFS state in userspace.
*   **CWD**: Current Working Directory tracking.
*   **Mount Table**: Per-process mount points.
*   **FD Table**: `Vec<FileEntry>` mapping `int fd` to `{ Capability, Offset, Flags }`.

### 3.2 Signal Emulation
*   **Receiver**: Registers an IPC endpoint with **Warren** to receive "Software Interrupts".
*   **Dispatcher**: When a signal IPC arrives, APE interrupts the main thread (or uses a helper thread) to execute the registered POSIX signal handler.

### 3.3 Process primitives
*   `fork()`: Implemented via **Warren** (creating a new task, COW memory).
*   `exec()`: Loading a new ELF via **Warren**.

## 4. Usage Modes

1.  **Native Link**: Glenda-aware Rust apps use `libglenda` directly (bypassing parts of APE, using native traits).
2.  **Legacy Link**: C/C++ apps link against `musl-glenda` (APE backend).
3.  **Binary Translation**: Pure Linux binaries run under **Tux**, where Tux acts as the "Server-side APE".

## 5. Performance
*   **Zero-Copy I/O**: APE uses shared memory for `read/write` payloads > 512 bytes to avoid IPC copying overhead.
*   **Batched Syscalls**: (Planned) Aggregating multiple small metadata ops.
