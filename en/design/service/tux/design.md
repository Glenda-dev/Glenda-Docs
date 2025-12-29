# Tux: POSIX Service Daemon (posixd)

Tux is the POSIX personality server for the Glenda microkernel. It provides a POSIX-compliant environment for user applications by mapping standard POSIX system calls to Glenda's microkernel primitives (Capabilities, IPC, Endpoints).

## 1. Core Responsibilities

- **Process Management**: Maintains the PID hierarchy, handles process creation (`fork`), execution (`exec`), and termination (`exit`, `wait`).
- **File Descriptor (FD) Management**: Manages per-process FD tables, mapping integer FDs to IPC Endpoint Capabilities pointing to Gopher (VFS) or Unicorn (Drivers).
- **Signal Handling**: Implements asynchronous signal delivery using Glenda's Notification objects.
- **System Call Translation**: Acts as the primary `ecall` handler (via IPC) for standard C libraries (libc), translating POSIX calls into microkernel IPC requests.

## 2. Architecture

Tux runs as a high-priority user-space server. It listens on a global Endpoint provided by the Root Task (9ball).

### 2.1 Identification via Badges
To ensure security, Tux relies on **IPC Badges**. When the Root Task or Tux itself distributes a session capability to a new process, it attaches a unique, immutable Badge to the Endpoint.
- When Tux receives a message, the kernel injects the Badge into the receiver's context.
- Tux uses this Badge to look up the `ProcessContext` in its internal process table, preventing processes from spoofing their PID.

## 3. Data Structures

### 3.1 IPC Protocol
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

#[repr(C)]
pub struct PosixRequest {
    pub syscall_id: PosixSyscall,
    pub args: [usize; 5],
}

#[repr(C)]
pub struct PosixResponse {
    pub error: i32,    // 0 for success, negative for errno
    pub ret: usize,    // Return value
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

## 4. Interaction Flows

### 4.1 File Open (`open`)
1. **App** calls `open("/etc/motd")` in libc.
2. **libc** sends a `PosixRequest` to Tux via IPC.
3. **Tux** receives the request and identifies the caller via its **Badge**.
4. **Tux** forwards the path lookup to **Gopher (VFS)**.
5. **Gopher** returns a new `Endpoint Capability` for the file session.
6. **Tux** allocates a new FD (e.g., `3`), stores the capability in the process's `fd_table`, and returns `3` to the App.

### 4.2 Process Creation (`fork`)
1. **App** calls `fork()`.
2. **Tux** requests the kernel to create a new TCB, CSpace, and VSpace.
3. **Tux** performs a deep copy of the parent's VSpace (or sets up Copy-on-Write).
4. **Tux** clones the parent's `fd_table` into the child.
5. **Tux** registers the new PID and returns the child PID to the parent, and `0` to the child.

## 5. Security and Isolation

- **Capability-Based**: Tux only grants FDs (Capabilities) that the process is authorized to access.
- **No Global State**: All POSIX state is encapsulated within Tux. The kernel remains unaware of "processes" or "files".
- **Resource Accounting**: Tux uses its own `Untyped` memory pool to allocate metadata for new processes, ensuring one process cannot exhaust system-wide memory by creating too many sub-processes.
