# Memory Management Design

## 1. Overview

In Glenda's microkernel architecture, the kernel's role in memory management is reduced to **Mechanism Only**: it ensures safety and isolation but dictates no policy. All physical memory allocation and virtual memory mapping policies are implemented in user space.

## 2. Physical Memory (Untyped)

*   **Boot**: All free physical memory is tracked as `Untyped` regions.
*   **Root Task**: Receives capabilities for all `Untyped` regions at startup.
*   **Allocation**: There is no global `kmalloc` for user objects. Userspace must "Retype" `Untyped` memory into specific kernel objects (`Frame`, `TCB`, `PageTable`, `Endpoint`, `CNode`) via system calls.
*   **Retype Operation**:
    *   Invoked on an `Untyped` capability.
    *   Splits a portion of the untyped memory to create new kernel objects.
    *   Returns capabilities to the new objects.

## 3. Virtual Memory (Address Spaces)

*   **VSpace**: An address space is represented by a top-level `PageTable` capability.
*   **UTCB Mapping**: Each thread has a UTCB page mapped into its VSpace. This mapping is established during thread creation/configuration.
*   **Mapping Operation**:
    *   Performed by invoking `sys_invoke` on a `PageTable` capability.
    *   **Arguments**: `FrameCap`, `vaddr`, `permissions`.
    *   The kernel simply inserts the PTE (Page Table Entry) into the hardware page table. It does not track VMAs (Virtual Memory Areas) or manage regions; that is the responsibility of the user-space pager.
*   **Unmapping Operation**:
    *   Performed by invoking `sys_invoke` on a `PageTable` capability.
    *   **Arguments**: `vaddr`.
    *   Removes the mapping.

## 4. Kernel Memory

*   The kernel maintains a small internal allocator for its own minimal metadata needs during boot.
*   Once the system is running, all memory required for kernel objects (TCBs, Page Tables, etc.) is derived from user-provided `Untyped` memory. This ensures strict resource accounting and prevents Denial of Service attacks against the kernel memory allocator.

## 5. Capability Interface

The `PageTable` kernel object exposes the following methods via `sys_invoke`:

| Method | Description |
| :--- | :--- |
| `Map` | Map a `Frame` capability to a virtual address. |
| `Unmap` | Unmap a virtual address. |
| `GetStatus` | Query the status of a mapping (optional). |

The `Untyped` kernel object exposes:

| Method | Description |
| :--- | :--- |
| `Retype` | Create new kernel objects from this memory region. |
