# Glenda Microkernel Migration Todo List

## Phase 0: Build System Infrastructure (Ref: `xtask.md`)
- [ ] **Dependencies**: Add `serde` and `toml` to `xtask/Cargo.toml`.
- [ ] **Configuration**: Create `components.toml` for defining boot modules.
- [ ] **Modular Refactoring**:
    - [ ] Split `xtask` into modules: `config`, `build`, `pack`, `qemu`, `fs`.
    - [ ] Implement `config.rs` to parse `components.toml`.
    - [ ] Implement `pack.rs` to generate `target/bootfs/payload.bin` (Header + Modules).
    - [ ] Implement `build.rs` to ensure `pack` runs before kernel build.
- [ ] **Kernel Integration (Incbin Strategy)**:
    - [ ] Create `kernel/src/asm/payload.S` using `.incbin` to include `payload.bin`.
    - [ ] Create `kernel/src/boot_payload.rs` to expose `_payload_start/end` symbols.
    - [ ] Update `kernel/build.rs` to watch `payload.bin` for changes.
- [ ] **GRUB / Multiboot Support**:
    - [ ] Add `multiboot2` header to kernel assembly entry.
    - [ ] Implement Multiboot2 tag parser in Rust.
    - [ ] Abstract `PayloadSource` trait (Embedded vs Multiboot).
    - [ ] Implement `xtask dist` to generate bootable ISO.

## Phase 1: Kernel Cleanup & Stripping (Ref: `mem.md`, `proc.md`)
- [ ] **Remove Kernel-Mode Drivers**:
    - [ ] Comment out `kernel/src/drivers`.
    - [ ] Move driver code to `staging/drivers` for later porting.
- [ ] **Remove Kernel-Mode Filesystem**:
    - [ ] Comment out `kernel/src/fs`.
    - [ ] Move FS code to `staging/fs`.
- [ ] **Simplify Memory Management**:
    - [ ] Remove `kernel/src/mem/{mmap.rs, uvm.rs, vm.rs}`.
    - [ ] Remove VMA (Virtual Memory Area) tracking from kernel.
    - [ ] Retain `pmem.rs` (Physical Frame Allocator) and `pagetable.rs`.
- [ ] **Simplify Process Management**:
    - [ ] Rename `Process` struct to `TCB` (Thread Control Block).
    - [ ] Remove `parent`, `exit_code`, `sleep_chan`, `mmap_head`.
    - [ ] Remove `fork`, `exec`, `wait` logic.

## Phase 2: Core Microkernel Mechanisms (Ref: `cap.md`, `ipc.md`)
- [ ] **Capability System (`kernel/src/cap.rs`)**:
    - [ ] Define `CapType` enum (`Untyped`, `TCB`, `Endpoint`, `Frame`, `PageTable`, `IrqHandler`, `CNode`).
    - [ ] Define `Capability` struct (object ref + permissions + badge).
    - [ ] Implement `CSpace` (Capability Space) using a radix tree or single-level table (`CNode`).
    - [ ] Implement `Cap::derive` (Mint) and `Cap::revoke`.
- [ ] **Thread Control Block (`kernel/src/proc/tcb.rs`)**:
    - [ ] Add `cspace_root` (Capability to root CNode).
    - [ ] Add `vspace_root` (Capability to root PageTable).
    - [ ] Add `irqhandler` (Capability to exception handler Endpoint).
    - [ ] Add `ipc_buffer` (User-mapped buffer for long messages).
    - [ ] Add `state` enum (`Running`, `Ready`, `BlockedSend`, `BlockedRecv`, `BlockedCall`).
- [ ] **IPC Subsystem (`kernel/src/ipc`)**:
    - [ ] Define `IPCMessage` format (Tag + Registers + Optional Buffer).
    - [ ] Implement `Endpoint` kernel object (queue of waiting TCBs).
    - [ ] Implement `Notification` kernel object (async signals/interrupts).
    - [ ] Implement synchronous transfer logic (direct context switch).

## Phase 3: System Call Interface (Ref: `syscall.md`)
- [ ] **Remove Legacy Syscalls**:
    - [ ] Delete `kernel/src/syscall/{fs,proc,brk,mmap}.rs`.
- [ ] **Implement Microkernel Syscalls**:
    - [ ] `sys_call(cptr, ...)`: Generic invocation / RPC.
    - [ ] `sys_reply_recv(cptr, ...)`: Server loop optimization.
    - [ ] `sys_send(cptr, ...)` / `sys_recv(cptr, ...)`.
    - [ ] `sys_yield()`.
    - [ ] `sys_debug_putc(char)` (Debug only).
- [ ] **Implement Object Invocations**:
    - [ ] `Untyped::Retype`: Create new objects from free memory.
    - [ ] `PageTable::Map/Unmap`: Manipulate address spaces.
    - [ ] `TCB::Configure/Resume`: Setup and start threads.
    - [ ] `CNode::Mint/Copy/Move`: Manage capabilities.

## Phase 4: Exception & Interrupt Handling (Ref: `irq.md`)
- [ ] **Exception Handling (Application Interrupts)**:
    - [ ] Modify `trap_user_handler` to stop handling PageFaults internally.
    - [ ] Construct `FaultMessage` (containing `scause`, `stval`, `sepc`).
    - [ ] Force-send message to `current_thread.irqhandler`.
- [ ] **Interrupt Handling**:
    - [ ] Disable in-kernel driver logic.
    - [ ] Map hardware IRQs to `Notification` objects.
    - [ ] On IRQ: Mask interrupt -> Signal Notification -> Ack from user -> Unmask.

## Phase 5: Root Task & Bootstrapping
- [ ] **Root Task Environment**:
    - [ ] Create `apps/root-task` (no_std binary).
    - [ ] Define `BootInfo` structure (passed to Root Task).
- [ ] **Kernel Boot Sequence**:
    - [ ] Initialize CPU and minimal memory.
    - [ ] Create initial `TCB` for Root Task.
    - [ ] **Cap Distribution**:
        - [ ] Create `Untyped` caps for all free RAM.
        - [ ] Create `IrqHandler` caps for all IRQs.
        - [ ] Create `Device` caps for MMIO regions.
    - [ ] Load Root Task ELF.
    - [ ] Switch to User Mode.

## Phase 6: User-Space Services
- [ ] **libglenda**:
    - [ ] Syscall wrappers (`seL4` style).
    - [ ] Runtime support (crt0, stack setup).
- [ ] **Serial Driver**:
    - [ ] User-space driver listening on Notification.
- [ ] **Memory Manager / Init**:
    - [ ] Simple allocator managing `Untyped` caps.
    - [ ] ELF Loader service.
