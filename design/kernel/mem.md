# Memory Management Migration Guide

## 1. Current Status
`kernel/src/mem` currently implements a full VM subsystem:
- `mmap`/`munmap` logic.
- VMA (Virtual Memory Area) tracking.
- `brk` heap management.
- Page table manipulation mixed with policy.

## 2. Target Architecture (Microkernel)
The kernel's role is reduced to **Mechanism Only**: it ensures safety and isolation but dictates no policy.

### 2.1 Physical Memory (Untyped)
- **Boot**: All free physical memory is tracked as `Untyped` regions.
- **Root Task**: Receives capabilities for all `Untyped` regions.
- **Allocation**: No global `kmalloc` for user objects. Userspace must "Retype" `Untyped` memory into specific kernel objects (`Frame`, `TCB`, `PageTable`) via system calls.

### 2.2 Virtual Memory (Address Spaces)
- **VSpace**: Represented by a top-level `PageTable` capability.
- **Mapping**:
    - No `mmap` syscall.
    - Instead: `invoke(PageTableCap, Map, FrameCap, vaddr, perms)`.
    - The kernel simply inserts the PTE. It does not track VMAs.
- **Unmapping**:
    - `invoke(PageTableCap, Unmap, vaddr)`.

### 2.3 Kernel Memory
- The kernel still needs a small internal allocator (`buddy.rs` / `slab`) for its own metadata (if not using a fully static partitioning model like seL4).
- Ideally, kernel metadata for user objects is stored in the memory provided by the user (via `Untyped` retyping), ensuring strict resource accounting.

### 2.4 Migration Steps
1.  Keep `pagetable.rs` (hardware logic).
2.  Keep `pmem.rs` (basic frame allocator for boot).
3.  **Delete** `mmap.rs`, `uvm.rs` (user logic), `vm.rs` (high-level logic).
4.  Implement `Retype` and `Map` invocations in `cap.rs` / `syscall.rs`.
