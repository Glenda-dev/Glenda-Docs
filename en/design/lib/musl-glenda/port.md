# musl-glenda Porting Design

## 1. Introduction

`musl-glenda` is the port of the [musl libc](https://musl.libc.org/) to the Glenda operating system. Since Glenda is a microkernel that does not implement the POSIX syscall ABI in the kernel, `musl-glenda` acts as the translation layer. It converts standard POSIX function calls (like `open`, `read`, `fork`) into IPC messages sent to the **Tux** POSIX Server.

## 2. Architecture

The core difference between `musl-glenda` and `musl` on Linux is the implementation of the system call mechanism.

### 2.1 Syscall Redirection

In standard Linux `musl`, system calls are performed using architecture-specific instructions (e.g., `ecall` on RISC-V, `syscall` on x86_64).

In `musl-glenda`, the `__syscall` function is overridden to perform a **Glenda IPC Call**.

*   **Location**: `arch/riscv64/syscall_arch.h` (and similar for other archs).
*   **Mechanism**:
    1.  Retrieve the **Tux Endpoint Capability** (stored in a global variable during startup).
    2.  Marshal the syscall number and arguments into the IPC message registers.
    3.  Invoke `seL4_Call` (or the Glenda equivalent `glenda_call`) to send the message to Tux.
    4.  Unpack the return value from the IPC response.

```c
// Conceptual implementation 
long __syscall_arch(long n, long a1, long a2, long a3, long a4, long a5, long a6) {
    glenda_ipc_msg_t msg;
    msg.label = n; // Syscall number
    msg.args[0] = a1;
    // ... pack args ...
    
    glenda_ipc_recv_t resp = glenda_call(GLOBAL_TUX_CAP, &msg);
    
    return resp.ret_val;
}
```

### 2.2 The "Tux" Connection

For `musl-glenda` to function, every process must have a capability to talk to the Tux server.

*   **Bootstrap**: When Tux (or the Init task) starts a new process, it passes the **Tux Session Capability** as a specific handle (e.g., handle index `1`) in the new process's CSpace.
*   **Initialization**: The `crt0` (C Runtime Start) code reads this capability and stores it in a hidden global variable `__glenda_tux_cap` accessible only to the syscall wrapper.

## 3. Key Subsystems

### 3.1 Process Startup (crt0)

The entry point `_start` performs the following:

1.  **Get Boot Info**: Reads the initial register state provided by the kernel/loader.
2.  **Setup TLS**: Initializes Thread Local Storage.
3.  **Locate Capabilities**: Identifies the `Tux` endpoint and `Factotum` endpoint (for memory management).
4.  **Initialize Heap**: Sets up the initial allocator state.
5.  **Call `__libc_start_main`**: Hands control to generic musl initialization and finally `main()`.

### 3.2 Memory Management (`mmap`, `brk`)

`musl` relies on `mmap` and `brk` for memory allocation.

*   **`brk`**: Not supported or emulated by allocating new pages contiguously.
*   **`mmap`**:
    *   **Anonymous**: Converted into an IPC call to **Factotum** (the Process Manager) to allocate backing frames and map them into the VSpace.
    *   **File-backed**: Converted into an IPC call to **Tux**, which coordinates with **Gopher** (VFS) and **Factotum** to map the file content.

### 3.3 Threading (`pthread`)

Glenda threads are kernel objects managed by Factotum.

*   **`pthread_create`**:
    1.  Allocates a stack and TLS for the new thread in the current VSpace.
    2.  Sends an IPC to **Factotum** requesting a new Thread Control Block (TCB).
    3.  Configures the new TCB to start executing at the thread entry point.
*   **Synchronization**: `futex` is implemented via kernel notifications or a dedicated kernel futex object if available. If not, it falls back to a user-space spinlock with `yield` (inefficient) or a blocking IPC to Factotum for wait queues.

### 3.4 Signal Handling

Signals are simulated using Glenda's **Notification Objects**.

1.  **Registration**: When `sigaction` is called, `musl-glenda` registers the handler locally and informs Tux via IPC.
2.  **Delivery**:
    *   Tux receives a hardware fault (from Factotum) or a `kill()` request.
    *   Tux sends a **Notification** to the target process's "Signal Notification Object".
    *   The target process has a dedicated high-priority "Signal Thread" waiting on this notification.
    *   When the notification triggers, the Signal Thread interrupts the main thread (or modifies its context) to execute the registered signal handler.

## 4. File I/O

All file descriptors (FDs) are virtual integers managed by Tux.

*   **`open`, `read`, `write`, `close`**:
    *   These are direct IPC calls to Tux.
    *   Tux validates the FD and forwards the data to the underlying service (e.g., Gopher for files, 9ball for logs).
    *   **Zero-Copy Optimization**: For large reads/writes, `musl-glenda` may use a shared memory buffer established with Tux/Gopher instead of copying data through IPC registers.

## 5. Roadmap

1.  **Phase 1**: Basic `write` (stdout) and `exit` support. (Hello World)
2.  **Phase 2**: `mmap` via Factotum (Heap allocation).
3.  **Phase 3**: `open`/`read` via Tux (File I/O).
4.  **Phase 4**: `fork`/`exec` support.
5.  **Phase 5**: Full `pthread` support.
