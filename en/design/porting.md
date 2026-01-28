# Glenda Kernel & Library Porting Guide

This document details the Hardware Abstraction Layer (HAL) interfaces that need to be implemented when porting Glenda and its core libraries to a new hardware architecture (e.g. LoongArch, AArch64, etc.).

## 1. Porting Overview

Glenda's core kernel logic is decoupled from low-level hardware details via HAL interfaces. The porting work is mainly divided into two parts:
1. **Kernel Porting**: Implement the interfaces defined in `kernel/src/hal/api` under `kernel/src/hal/arch/<new_arch>`.
2. **Library Porting**: Implement architecture-specific system calls and runtime support in `lib/libglenda-rs/src/arch/<new_arch>.rs`.

### Kernel Directory Structure

The recommended directory structure is as follows (using `loongarch64` as an example):

```
kernel/src/hal/loongarch64/
├── mod.rs          # Module export and architecture mounting
├── boot.rs/S       # Boot and boot detection
├── console.rs      # Early serial console
├── cpu.rs          # CPU core capabilities
├── irq.rs          # Interrupt management
├── mem.rs          # Memory management and page tables
├── platform.rs     # Platform-level functions
├── proc.rs         # Process/Thread context
├── runtime.rs      # Runtime assistance (backtrace)
└── trap.rs         # Exception context and handling
```

## 2. Kernel Interface Implementation Checklist (`kernel/src/hal/api`)

You need to implement all modules defined in `kernel/src/hal/api`.

### 2.1 Boot - `interface/boot.rs`

- **Responsibility**: Detect bootloader type and platform information.
- **Functions**:
    - `detect() -> (usize, PlatformInfo)`: Returns the CPUID and platform information structure passed during the boot stage.
    - Need to define the assembly entry point `_start` and handle multi-core boot.

### 2.2 Memory Management - `interface/mem.rs`

This is the most complex part of porting, involving paging mechanisms and address space layout.

#### Required Constants
- `PGSIZE`: Page size.
- `VA_MAX`: Virtual address space limit.
- `PHYS_MAP_BASE`: Physical memory linear mapping base address.
- `KERNEL_BASE`: Kernel link base address.
- `PT_LEVELS`: Number of page table levels.
- `PGNUM`: Number of page table entries per level.

#### Required Structs & Methods
- `Pte`: Page table entry. Must implement `null`, `from`, `as_usize`, `get_ppn`, `set_ppn`, `get_flags`, `set_flags`, `is_valid`.
- `PteFlags`: Flag bits.
- `PageTable`: Page table structure, containing `entries: [Pte; PGNUM]`.
- `perms` module: Defines constants like `VALID`, `READ`, `WRITE`, `EXECUTE`, `USER`, `GLOBAL`, `ACCESSED`, `DIRTY`, etc.

#### Required Functions
- `flush_tlb(vaddr: Option<VirtAddr>)`: Flush TLB.
- `activate_pagetable(root_paddr: PhysAddr)`: Activate page table.
- `deactivate_pagetable()`: Disable paging.
- `get_vpn_index(va, level) -> VPN`: Get the index of a virtual address in a certain page table level.
- `kpt_setup(kpt: &mut PageTable)`: Setup kernel page table (e.g. mapping kernel segments).

### 2.3 Exceptions & Traps - `interface/trap.rs`

#### Structs
- `TrapContext`: Saves register state.
    - `set_return_value(value)`
    - `get_syscall_args() -> (usize, usize)`
    - `from_trapframe(tf)`
- `TrapFrame`: Trap frame.
    - `set_badge(badge)`
    - `get_epc()`, `set_epc(epc)`
    - `set_tf(addr)`
    - `update_context(ctx)`
    - `configure_kernel(...)`: Configure kernel mode return information.

#### Functions
- `vector_init()`: Initialize exception vector table.
- `get_cause() -> TrapCause`: Get exception cause.
- `get_pc() -> usize`: Get PC at the time of exception.
- `advance_pc(pc, offset)`: Move PC.
- `get_address() -> VirtAddr`: Get fault address (e.g. PageFault address).
- `get_status() -> usize`: Get status register.

### 2.4 Interrupt Management (IRQ) - `interface/irq.rs`

#### CPU Local State
- `enable()`, `disable()`: Enable/Disable interrupts.
- `is_enabled()`: Query status.
- `wfi()`: Wait for interrupt.

#### Platform Controller
- `MAX_IRQS`: Maximum number of interrupts.
- `init()`: Controller initialization.
- `init_cpu(cpuid)`: Current core initialization.
- `mask(irq, cpuid)`, `unmask(irq, cpuid)`: Mask/Unmask.
- `claim(cpuid) -> Option<u32>`: Get pending interrupt number.
- `complete(irq, cpuid)`: Send EOI.
- `set_affinity`, `set_priority`: Set affinity and priority.
- `clear_soft()`: Clear software interrupt.

### 2.5 CPU Core Capabilities - `interface/cpu.rs`

- `MAX_CPUS`: Maximum supported cores.
- `cpu_id()`: Get current core ID.
- `read_cycle()`, `read_time()`: Read counters.
- `timer_set_next(next)`: Set timer interrupt.

### 2.6 Process Context - `interface/proc.rs`

- `ProcContext`: Thread context.
    - `new()`
    - `configure(entry, stack_top)`
    - `set_fp(fp)`
- `switch_context(old, new)`: Context switch assembly implementation.

### 2.7 Platform Functions - `interface/platform.rs`

- `PlatformInfo`: Platform information structure.
- `init(dtb)`: Platform initialization.
- `bootargs()`: Get boot arguments.
- `shutdown()`, `reboot()`: Shutdown and reboot.
- `send_ipi(mask)`: Send Inter-Processor Interrupt.
- `memory_range()`, `range()`: Get memory range.
- `initrd()`: Get InitRD range.
- `bootstrap_cpus()`: Boot secondary cores.

### 2.8 Console - `interface/console.rs`

- `init()`: Initialize.
- `print(args)`: Print formatted string.

### 2.9 Runtime - `interface/runtime.rs`

- `backtrace()`: Print call stack.

## 3. Library Interface Implementation Checklist (`lib/libglenda-rs`)

Implement the interfaces defined in `lib/libglenda-rs/src/arch/api.rs` inside `lib/libglenda-rs/src/arch/<arch>.rs`, and export them in `lib/libglenda-rs/src/arch/mod.rs`.

#### Required Functions
- `unsafe fn syscall(cptr: usize, method: usize) -> usize`: Execute system call.
- `unsafe fn syscall_recv(cptr: usize, method: usize) -> (usize, usize)`: Execute receive system call (returns two values).
- `unsafe fn panic_break()`: Halt or breakpoint due to panic.
- `fn backtrace()`: User mode backtrace.

## 4. Suggested Porting Steps

1.  **Establish Skeleton**: Create directory structure, fill all interfaces with `unimplemented!()`.
2.  **Boot Code**: Write boot assembly and `boot.rs`, implement `console` output.
3.  **Memory Initialization**: Implement `mem` module, configure page tables.
4.  **Exceptions & Interrupts**: Implement `trap`, `irq` modules.
5.  **Process Switching**: Implement context switching in `proc` module.
6.  **Platform Functions**: Perfect `platform` module to support multi-core and device tree parsing.
7.  **Library Support**: Implement syscall assembly in `libglenda-rs`.
