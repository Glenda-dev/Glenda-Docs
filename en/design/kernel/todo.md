# Glenda Microkernel Migration Todo List

## Phase 0: Build System Infrastructure (Ref: `xtask.md`)
- [x] **Dependencies**: Add `serde` and `toml` to `xtask/Cargo.toml`.
- [x] **Configuration**: Create `config.toml` for defining boot modules.
- [x] **Modular Refactoring**:
    - [x] Split `xtask` into modules: `config`, `build`, `pack`, `qemu`, `fs`.
    - [x] Implement `config.rs` to parse `config.toml`.
    - [x] Implement `pack.rs` to generate `target/bootfs/payload.bin` (Header + Modules).
    - [x] Implement `build.rs` to ensure `pack` runs before kernel build.
- [x] **Kernel Integration (Incbin Strategy)**:
    - [x] Create `kernel/src/asm/payload.S` using `.incbin` to include `payload.bin`.
    - [x] Update `kernel/build.rs` to watch `payload.bin` for changes.
- [ ] **GRUB / Multiboot Support**:
    - [ ] Add `multiboot2` header to kernel assembly entry.
    - [ ] Implement Multiboot2 tag parser in Rust.
    - [ ] Abstract `PayloadSource` trait (Embedded vs Multiboot).
    - [ ] Implement `xtask dist` to generate bootable ISO.

## Phase 1: Kernel Cleanup & Stripping (Ref: `mem.md`, `proc.md`)
- [x] **Remove Kernel-Mode Drivers**:
    - [x] Comment out `kernel/src/drivers`.
    - [x] Move driver code to `staging/drivers` for later porting.
- [x] **Remove Kernel-Mode Filesystem**:
    - [x] Comment out `kernel/src/fs`.
    - [x] Move FS code to `staging/fs`.
- [x] **Simplify Memory Management**:
    - [x] Remove `kernel/src/mem/{mmap.rs, uvm.rs}`.
    - [x] Remove VMA (Virtual Memory Area) tracking from kernel.
    - [x] Retain `pmem.rs` (Physical Frame Allocator) and `pagetable.rs`.
- [x] **Simplify Process Management**:
    - [x] Rename `Process` struct to `TCB` (Thread Control Block).
    - [x] Remove `parent`, `exit_code`, `sleep_chan`, `mmap_head`.
    - [x] Remove `fork`, `exec`, `wait` logic.

## Phase 2: Core Microkernel Mechanisms (Ref: `cap.md`, `ipc.md`)
- [x] **Capability System (`kernel/src/cap.rs`)**:
    - [x] Define `CapType` enum (`Untyped`, `TCB`, `Endpoint`, `Frame`, `PageTable`, `IrqHandler`, `CNode`, `Reply`).
    - [x] Define `Capability` struct (object ref + permissions + badge).
    - [x] Implement `CSpace` (Capability Space) using a radix tree or single-level table (`CNode`).
    - [x] Implement `Cap::derive` (Mint) and `Cap::revoke`.
- [x] **Thread Control Block (`kernel/src/proc/tcb.rs`)**:
    - [x] Add `cspace_root` (Capability to root CNode).
    - [x] Add `vspace_root` (Capability to root PageTable).
    - [x] Add `fault_handler` (Capability to exception handler Endpoint).
    - [x] Add `ipc_buffer` (User-mapped buffer for long messages).
    - [x] Add `state` enum (`Running`, `Ready`, `BlockedSend`, `BlockedRecv`, `BlockedCall`).
- [x] **IPC Subsystem (`kernel/src/ipc`)**:
    - [x] Define `IPCMessage` format (Tag + Registers + Optional Buffer).
    - [x] Implement `Endpoint` kernel object (queue of waiting TCBs).
    - [x] Implement `Notification` kernel object (via `Endpoint` with `notification_word`).
    - [x] Implement synchronous transfer logic (direct context switch).

## Phase 3: System Call Interface (Ref: `syscall.md`)
- [x] **Remove Legacy Syscalls**:
    - [x] Delete `kernel/src/syscall/{fs,proc,brk,mmap}.rs`.
- [x] **Implement Microkernel Syscalls**:
    - [x] `sys_invoke(cptr, ...)`: Generic invocation.
- [x] **Implement Object Invocations**:
    - [x] `Untyped::Retype`: Create new objects from free memory.
    - [x] `PageTable::Map/Unmap`: Manipulate address spaces.
    - [x] `TCB::Configure/Resume`: Setup and start threads.
    - [x] `CNode::Mint/Copy/Move`: Manage capabilities.
    - [x] `Reply::Reply`: Reply to a Call.

## Phase 4: Interrupts & Timer (Refining)
- [ ] **Fix S-Mode Timer**:
    - [ ] Remove M-Mode CSR access (`mscratch`, `mtimecmp`) from `kernel/src/trap/handler/vector.rs`.
    - [ ] Use SBI calls (`sbi_set_timer`) for timer management.
- [ ] **Interrupt Handling**:
    - [ ] Implement `IrqHandler` capability invocation (Ack, SetNotification).
    - [ ] Map hardware IRQs to `Notification` objects.

## Phase 5: Root Task (9ball) Implementation
- [ ] **Project Structure**:
    - [ ] Initialize `9ball` as a `no_std` binary.
    - [ ] Ensure it links against `libglenda`.
- [ ] **Bootstrapping**:
    - [ ] Implement `BootInfo` parsing in user-space (match kernel layout).
    - [ ] Implement a basic physical memory allocator (managing `Untyped` caps).
    - [ ] Implement a basic CSpace manager (tracking slots).

## Phase 6: User-Space Foundation (libglenda)
- [x] **Runtime Support**:
    - [x] Implement `crt0` (entry point, stack setup).
    - [x] Implement `panic_handler` for user space.

## Phase 7: Basic Services
- [ ] **Serial Driver**:
    - [ ] User-space driver listening on Notification from UART IRQ.
- [ ] **Timer Driver**:
    - [ ] User-space driver managing timer events.

## Phase 8: POSIX & Shell
- [ ] **Tux Server**:
    - [ ] Basic POSIX signal handling.
    - [ ] `fork`/`exec` emulation via TCB manipulation.
- [ ] **Shell**:
    - [ ] Basic command line interface.
