# Process Management Migration Guide

## 1. Current Status
The current `Process` struct in `kernel/src/proc/process.rs` is a monolithic design, containing:
- Memory management fields (`mmap_head`, `heap_top`, `heap_base`).
- Process relationship fields (`parent`, `exit_code`).
- File system related fields (implied by `fs_init_wrapper`).

## 2. Target Architecture (Microkernel)
In the L4 architecture, we strictly separate the **Execution Unit (Thread)** from the **Resource Container (Namespace)**.

### 2.1 Separation of Concerns
*   **Namespace (Protection Domain)**: Defines *what* can be accessed.
    *   **CSpace (Capability Space)**: The namespace for kernel objects. Represented by a root `CNode`.
    *   **VSpace (Address Space)**: The namespace for memory addresses. Represented by a root `PageTable`.
*   **Thread (TCB)**: Defines *how* code executes.
    *   Has its own register state, priority, and time slice.
    *   **References** a CSpace and VSpace, but does not "own" them structurally. Multiple threads can share the same CSpace/VSpace (multithreading).

### 2.2 Data Structure Changes
Rename `Process` to `TCB` (Thread Control Block) and strip it down.

**Remove:**
- `mmap_head`, `heap_top`, `heap_base`: Memory layout is managed by the user-space Pager/Loader.
- `parent`, `exit_code`, `sleep_chan`: Process hierarchy is managed by the Root Task or a Process Server.
- `name`: Can be kept for debugging, but strictly optional.
- `root_pt_pa`: Moved to VSpace definition.

**Add:**
- `cspace_root: Capability`: Cap pointing to the root `CNode` (defines the thread's capability view).
- `vspace_root: Capability`: Cap pointing to the root `PageTable` (defines the thread's address space).
- `utcb: VirtAddr`: Pointer to the User Thread Control Block (user-mapped kernel info area).
- `irqhandler: Capability`: The endpoint to send application interrupt (exception) IPCs to.
- `ipc_buffer: VirtAddr`: User-space buffer for long IPC messages.
- `state`: Update enum to include IPC states (`BlockedSend`, `BlockedRecv`, `BlockedCall`).

### 2.3 Scheduler Changes
- **Preemptive Priority Scheduling**: Implement strict priority-based scheduling (0-255).
- **Time Slices**: Round-robin for threads with equal priority.
- **IPC Integration**:
    - When `sys_recv` blocks, remove TCB from run queue, set state to `BlockedRecv`.
    - When `sys_send` blocks (if receiver not ready), set state to `BlockedSend`.
    - Direct switch: If possible, switch directly to the target thread on IPC to preserve cache locality.

### 2.3 Thread Creation
- Kernel no longer has `fork()` or `exec()`.
- New flow:
    1.  Parent thread invokes `Untyped` cap to retype memory into a new `TCB` object.
    2.  Parent invokes `TCB` cap to set registers (PC, SP) and CSpace.
    3.  Parent invokes `TCB` cap to `Resume` (start) the thread.
