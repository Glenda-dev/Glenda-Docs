# Glenda 内核移植指南

本文档详细说明了将 Glenda 内核移植到新硬件架构（例如 LoongArch, AArch64 等）时，需要实现的硬件抽象层 (HAL) 接口。

## 1. 移植概览

Glenda 的内核核心逻辑与底层硬件细节通过 `kernel/src/hal/interface` 目录下定义的接口进行解耦。移植工作的核心是在 `kernel/src/hal/arch/<new_arch>` 目录下实现这些接口，并在 `kernel/src/hal/mod.rs` 中进行挂载。

### 目录结构

建议的目录结构如下（以 `loongarch64` 为例）：

```
kernel/src/hal/loongarch64/
├── mod.rs          # 模块导出与架构挂载
├── boot.rs/S       # 启动汇编和初始化
├── console.rs      # 早期串口控制台
├── cpu.rs          # CPU 核心能力 (CpuHal)
├── irq.rs          # 中断管理 (IrqHal)
├── mem.rs          # 内存管理 (MemHal)
├── platform.rs     # 平台级功能 (PlatformHal)
├── proc.rs         # 进程/线程上下文 (ProcHal)
└── trap.rs         # 异常上下文与处理 (TrapHal)
```

## 2. 接口实现清单

你需要实现 `kernel/src/hal/interface` 中定义的所有模块。

### 2.1 启动 (Boot) - `interface/boot.rs`

这部分通常由汇编实现。你需要定义全局符号 `_start` 作为链接器入口。

- **职责**：
    - 初始化 CPU 状态。
    - 设置启动栈 (Boot Stack)。
    - 跳转到 `rust_main` (或类似名称的 Rust 入口)。
    - 处理多核启动同步（如果支持 SMP）。

### 2.2 内存管理 (Memory) - `interface/mem.rs`

这是移植中最复杂的部分，涉及分页机制和地址空间布局。

#### 必需常量
- `PGSIZE`: 页面大小（通常 4096）。
- `VA_MAX`: 虚拟地址空间上限。
- `PHYS_MAP_BASE`: 物理内存线性映射基地址 (HHDM)。
- `KERNEL_BASE`: 内核链接基地址。
- `PT_LEVELS`: 页表层级数。

#### 必需结构体
- `Pte`: 页表项，需实现一系列位操作方法 (`is_valid`, `is_leaf`, `set_ppn` 等)。
- `PteFlags`: 页表项标志位 (`VALID`, `READ`, `WRITE`, `EXECUTE`, `USER`, `GLOBAL`, `ACCESSED`, `DIRTY`)。
- `PageTable`: 页表结构。

#### 必需函数
- `flush_tlb(vaddr)`: 刷新 TLB，支持特定地址或全局刷新。
- `activate_pagetable(root_paddr)`: 写入页表寄存器 (如 `satp`, `ttbr0`)。
- `deactivate_pagetable()`: 禁用分页（或重置为初始状态）。

### 2.3 异常与陷阱 (Trap) - `interface/trap.rs`

负责处理用户态到内核态的切换，以及异常分发。

#### 必需结构体
- `TrapContext`: 保存通用寄存器、CSR 状态的结构体。

#### 必需函数
- `vector_init()`: 初始化异常向量表基地址 (`stvec`, `vbar_el1`)。
- `get_cause(&ctx)`: 解析异常原因，返回通用的 `TrapCause` 枚举。
- `get_pc(&ctx)`: 获取异常发生时的指令地址。
- `advance_pc(&mut ctx, bytes)`: 移动 PC（用于系统调用返回）。

### 2.4 中断管理 (IRQ) - `interface/irq.rs`

分为 **CPU 本地状态** 和 **平台控制器** 两部分。

#### CPU 本地状态
- `enable()` / `disable()`: 全局开关中断。
- `is_enabled()`: 查询当前中断状态。
- `wfi()`: 等待中断 (Wait For Interrupt)。

#### 平台控制器 (PLIC/GIC/IOCSR)
- `init()`: 初始化全局中断控制器。
- `init_cpu(cpuid)`: 初始化当前核的中断接口。
- `mask(irq, cpuid)` / `unmask(irq, cpuid)`: 屏蔽/解除屏蔽特定中断。
- `claim(cpuid)`: 获取当前挂起的中断号。
- `complete(irq, cpuid)`: 发送中断完成信号 (EOI)。

### 2.5 CPU 核心能力 - `interface/cpu.rs`

- `cpu_id()`: 获取当前硬件线程 ID (Hart ID)。
- `read_cycle()`: 读取性能计数器。
- `read_time()`: 读取实时时间（用于调度）。
- `timer_set_next(next)`: 设置下一次时钟中断的时间点。

### 2.6 进程上下文 (Process) - `interface/proc.rs`

负责内核线程之间的切换 (`switch`)。

#### 必需结构体
- `ProcContext`: 保存 Callee-saved 寄存器 (`ra`, `sp`, `s0-s11` 等)。

#### 必需函数
- `switch_context(old, new)`: 汇编实现的上下文切换。
- `ProcContext::new(entry, stack_top)`: 构造新线程的上下文。

### 2.7 平台功能 (Platform) - `interface/platform.rs`

- `bootargs()`: 获取启动参数（命令行）。
- `shutdown()` / `reboot()`: 关机与重启。
- `send_ipi(mask)`: 发送核间中断。
- `memory_range()`: 获取可用物理内存范围。

### 2.8 早期调试控制台 (Console) - `interface/console.rs`

- `init()`: 初始化早期串口（如 UART, SBI Console）。
- `print(args)`: 打印格式化字符串，用于内核启动早期的日志输出（Panic, Logs）。

## 3. 移植步骤建议

1.  **建立骨架**：创建目录结构，使用 `unimplemented!()` 填充所有接口，确保能通过编译。
2.  **启动代码**：编写 `boot.S`，能够打印字符到串口（验证 `console` 模块）。
3.  **内存初始化**：实现 `mem` 模块，配置页表，开启 MMU。
4.  **异常处理**：实现 `trap` 模块，能够捕获异常并打印寄存器。
5.  **支持中断**：实现 `irq` 和 `cpu` 模块，开启时钟中断。
6.  **多任务**：实现 `proc` 模块，支持 `switch_context`，启动第一个内核线程。
