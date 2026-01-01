# Factotum Design Document

## 1. Introduction
Factotum is the central resource and task manager of the Glenda microkernel system. While the kernel provides mechanisms (capabilities, threads, address spaces), Factotum provides the *policy* and high-level management. It acts as the "kernel" of the user space.

## 2. Responsibilities

*   **Memory Management**:
    *   Receives the bulk of system memory (Untyped) from 9Ball.
    *   Manages the global frame allocator.
    *   Handles page faults for user processes (lazy allocation, COW).
*   **Process & Thread Management**:
    *   **ELF Loading**: Parses ELF binaries and sets up the initial VSpace (Virtual Address Space).
    *   **Scheduling Policy**: While the kernel schedules threads, Factotum manages thread priorities and time slice allocation (via kernel capability manipulation).
    *   **Lifecycle**: Handles process creation (`spawn`) and destruction (`kill`).
    *   **Multi-threading**:
        *   Manages thread groups (all threads sharing the same VSpace).
        *   Implements thread-local storage (TLS) setup.
        *   Handles thread synchronization primitives (futex-like mechanism).
*   **Exception Handling**:
    *   Registered as the exception handler for all processes it spawns.
    *   Handles faults like segmentation faults or illegal instructions, potentially converting them into signals (for Tux) or error reports.
*   **Capability Management**:
    *   Acts as a capability broker, minting and distributing capabilities to new processes.

## 3. Architecture

Factotum runs as a high-priority service. It maintains a table of all running processes (Process Control Blocks - PCBs) in its own address space.

### Interaction with 9Ball
9Ball bootstraps Factotum. Once running, Factotum takes over the responsibility of creating future processes requested by 9Ball or other services.

### Interaction with Clients
Clients (like Tux or Gopher) request Factotum to:
*   `yield`: Give up CPU.
*   `map_memory`: Request shared memory.
*   `spawn_thread`: Create a new thread within an existing PD.

## 4. Interfaces

Factotum exposes its functionality via the **Factotum Protocol** (ID `0x0100`).

### 4.1 Process Lifecycle
*   `SPAWN(name_ptr, name_len, flags) -> [pid]`
    *   Creates a new process from an executable file.
    *   **Flags**: Controls behavior (e.g., `WAIT`, `DEBUG`, `RFNOMNT`).
        *   `RFNOMNT`: If set, requests Gopher to create a new, private namespace for the child process (Plan 9 style).
*   `EXIT(status) -> !`
    *   Terminates the calling process and notifies the parent.
*   `WAIT(pid) -> [status]`
    *   Blocks until the specified child process terminates.
    *   If `pid` is -1, waits for any child.
*   `KILL(pid, signal) -> [result]`
    *   Sends a termination signal to a target process.
*   `FORK() -> [pid]`
    *   Creates a copy of the calling process (COW semantics).


### 4.2 Memory Management
*   `SBRK(increment) -> [new_brk]`
    *   Adjusts the process's heap size (program break).
*   `MMAP(addr, len, prot, flags, fd, offset) -> [vaddr]`
    *   Maps files or anonymous memory into the address space.
    *   Supports shared memory creation.
*   `MUNMAP(addr, len) -> [result]`
    *   Unmaps a memory region.

### 4.3 Thread Control
*   `THREAD_CREATE(entry, stack, arg, tls_base) -> [tid]`
    *   Creates a new thread within the current process.
    *   Sets up the initial execution context and TLS.
*   `THREAD_EXIT(status) -> !`
    *   Terminates the calling thread.
*   `THREAD_JOIN(tid) -> [status]`
    *   Waits for a specific thread in the same process to exit.
*   `FUTEX_WAIT(addr, val, timeout) -> [result]`
    *   Atomically checks if `*addr == val`, and if so, sleeps.
*   `FUTEX_WAKE(addr, count) -> [woken_count]`
    *   Wakes up `count` threads waiting on `addr`.
*   `YIELD() -> []`
    *   Voluntarily gives up the remaining time slice.
*   `SLEEP(milliseconds) -> []`
    *   Suspends execution for a specified duration.

### 4.4 Debugging & Inspection
*   `GET_PID() -> [pid]`
    *   Returns the caller's Process ID.
*   `GET_PPID() -> [ppid]`
    *   Returns the parent's Process ID.
*   `PS() -> [buffer_cap]`
    *   Returns a read-only shared memory buffer containing the process list snapshot.

### 4.5 Capability Brokerage
*   `REQUEST_CAP(service_id) -> []` (Cap transfer via IPC)
    *   Requests a handle to a system service (e.g., Gopher, Rio).
    *   Factotum verifies permissions before granting.
