# Tux Design Document

## 1. Introduction
Tux is the POSIX Server for Glenda. Since Glenda is a microkernel with its own native async IPC, it does not natively support the POSIX API (fork, exec, blocking I/O, signals). Tux bridges this gap by providing a POSIX-compliant environment for user applications.

## 2. Responsibilities

*   **POSIX API Implementation**:
    *   Provides the standard C library interface (via `musl-glenda` which talks to Tux).
    *   Handles `fork()`, `exec()`, `wait()`, `pipe()`, `kill()`.
*   **Signal Handling**:
    *   Simulates Unix signals. Factotum notifies Tux of hardware faults, and Tux delivers them as signals (SIGSEGV, SIGILL) to the registered process.
*   **Process Group Management**:
    *   Manages PIDs, PGIDs, Sessions.
*   **File Descriptor Table**:
    *   Maintains the mapping between integer FDs and Glenda Capability Handles (for Gopher files, sockets, pipes).

## 3. Architecture

Tux runs as a high-priority user-space server. Applications linked with `musl-glenda` (libc) translate syscall instructions into IPC calls to Tux.

### Identification via Badges
To ensure security, Tux relies on **IPC Badges**. When Tux distributes a session capability to a new process, it attaches a unique, immutable Badge to the Endpoint.
- When Tux receives a message, the kernel injects the Badge into the receiver's context.
- Tux uses this Badge to look up the `ProcessContext` in its internal process table, preventing processes from spoofing their PID.

### The "Fork" Problem
`fork()` is difficult in microkernels. Tux works with Factotum to implement Copy-On-Write (COW) forking:
1.  Tux requests Factotum to clone the address space (marking pages read-only).
2.  Tux duplicates the file descriptor table.
3.  Tux creates the new thread via Factotum.

## 4. Interfaces

*   **Interface**: `org.glenda.posix.Tux`
*   **Methods**:
    *   `Syscall(num: u32, args: [u64; 6]) -> u64`
    *   `RegisterSignalHandler(signum: u32, handler: u64)`
    *   `GetPid() -> u32`

## 5. IPC Protocol
Defined in `libglenda-rs` for shared use between Tux and applications.

```rust
#[repr(u32)]
pub enum PosixSyscall {
    Open = 1,
    Read = 2,
    Write = 3,
    Close = 4,
    Fork = 5,
    Exec = 6,
    Wait = 7,
    Exit = 8,
    GetPid = 9,
}
```

### 3.2 Internal Process Tracking
Tux maintains a private table of all active POSIX processes.

```rust
struct PosixProcess {
    pid: i32,
    parent_pid: i32,
    // Microkernel Capabilities
    tcb_cap: CapPtr,
    vspace_cap: CapPtr,
    cspace_cap: CapPtr,
    // FD Table: FD -> Remote Service Endpoint
    fd_table: BTreeMap<i32, CapPtr>,
    // Signal State
    signal_notif: CapPtr,
    pending_signals: u64,
}
```


## 4. Security and Isolation

- **Capability-Based**: Tux only grants FDs (Capabilities) that the process is authorized to access.
- **No Global State**: All POSIX state is encapsulated within Tux. The kernel remains unaware of "processes" or "files".
- **Resource Accounting**: Tux uses its own `Untyped` memory pool to allocate metadata for new processes, ensuring one process cannot exhaust system-wide memory by creating too many sub-processes.
