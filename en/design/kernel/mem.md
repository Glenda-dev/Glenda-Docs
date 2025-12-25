# Memory Management Design

## 1. Overview

In Glenda's microkernel architecture, the kernel's role in memory management is reduced to **Mechanism Only**: it ensures safety and isolation but dictates no policy. All physical memory allocation and virtual memory mapping policies are implemented in user space.

## 2. Physical Memory (Untyped)

In Glenda, all physical memory is managed through the **Capability System**. There is no traditional page allocator (like a buddy system) accessible to user space. Instead, memory is represented as **Untyped Capabilities**.

*   **Boot**: The kernel identifies all available physical memory regions from the device tree (DTB) or multiboot info and wraps them into `Untyped` capabilities.
*   **Root Task**: At startup, the Root Task is granted capabilities for all `Untyped` regions. It acts as the system's primary memory manager.
*   **Retype Operation**: This is the only way to create new kernel objects.
    *   A user invokes `Retype` on an `Untyped` capability.
    *   The kernel carves out a contiguous block of memory from the `Untyped` region.
    *   The block is transformed into a specific type (e.g., `TCB`, `CNode`, `Frame`).
    *   A new capability to the created object is returned to the user.
*   **Memory Unification**: The internal `PhysFrame` abstraction is unified with the capability system. Once a frame is allocated to a capability, its lifecycle is governed by that capability's reference count.

## 3. Lifecycle and Reclamation

Glenda uses **Reference Counting** combined with a **Capability Derivation Tree (CDT)** to manage memory safety.

*   **Reference Counting**: Each kernel object (CNode, TCB, Endpoint, etc.) has a reference count.
    *   `Capability::clone()` increments the count.
    *   `Capability::drop()` decrements the count.
*   **Reclamation**: When the reference count of a capability reaches zero:
    *   The kernel triggers the object's destructor.
    *   If the object was a container (like a `CNode`), it recursively drops all capabilities it held.
    *   The underlying physical memory is returned to the `Untyped` pool or marked as free in the physical memory manager.
*   **Revocation**: By deleting a parent capability in the CDT, the kernel can recursively invalidate all derived child capabilities, ensuring that memory can be forcibly reclaimed by the resource owner.

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

## 5. Kernel Memory

*   **Static Memory**: The kernel image and initial data structures (like the early boot allocator) occupy a fixed portion of memory.
*   **Dynamic Objects**: Once the Root Task is running, the kernel **never** allocates memory for itself. Every TCB, Page Table, or CNode required by a process must be provided by that process (or its parent) via the `Retype` mechanism.
*   **Strict Accounting**: This design ensures that a malicious process cannot exhaust kernel memory. If a process runs out of its own `Untyped` quota, it simply cannot create more kernel objects.

## 6. Capability Interface

The `PageTable` kernel object exposes the following methods via `sys_invoke`:

| Method | Description |
| :--- | :--- |
| `Map` | Map a `Frame` capability to a virtual address. |
| `Unmap` | Unmap a virtual address. |
| `GetStatus` | Query the status of a mapping (permissions, dirty bit). |

The `Untyped` kernel object exposes:

| Method | Description |
| :--- | :--- |
| `Retype` | Create new kernel objects (TCB, CNode, Frame, etc.) from this region. |

The `CNode` kernel object (representing CSpace) exposes:

| Method | Description |
| :--- | :--- |
| `Copy` | Create a new capability to the same object (increments ref count). |
| `Mint` | Create a new capability with reduced rights or a new Badge. |
| `Move` | Move a capability from one slot to another. |
| `Revoke` | Recursively delete all capabilities derived from the target. |
| `Delete` | Remove a capability from a slot (decrements ref count). |
