# Glenda 内核与库移植指南

本文档详细说明了将 Glenda 及其核心库移植到新硬件架构（例如 LoongArch, AArch64 等）时，需要实现的硬件抽象层 (HAL) 接口。

## 1. 移植概览

Glenda 的核心逻辑与底层硬件细节通过 HAL 接口进行解耦。移植工作主要分为两部分：
1. **内核移植**: 在 `kernel/src/hal/arch/<new_arch>` 下实现 `kernel/src/hal/api` 定义的接口。
2. **库移植**: 在 `lib/libglenda-rs/src/arch/<new_arch>.rs` 中实现架构相关的系统调用与运行时支持。

### 内核目录结构

建议的目录结构如下（以 `loongarch64` 为例）：

```
kernel/src/hal/loongarch64/
├── mod.rs          # 模块导出与架构挂载
├── boot.rs/S       # 启动与引导检测
├── console.rs      # 早期串口控制台
├── cpu.rs          # CPU 核心能力
├── irq.rs          # 中断管理
├── mem.rs          # 内存管理与页表
├── platform.rs     # 平台级功能
├── proc.rs         # 进程/线程上下文
├── runtime.rs      # 运行时辅助 (backtrace)
└── trap.rs         # 异常上下文与处理
```

## 2. 内核接口实现清单 (`kernel/src/hal/api`)

你需要实现 `kernel/src/hal/api` 中定义的所有模块。

### 2.1 启动 (Boot) - `interface/boot.rs`

- **职责**：探测引导加载程序类型和平台信息。
- **函数**:
    - `detect() -> (usize, PlatformInfo)`: 返回启动阶段传递的CPUID和平台信息结构。
    - 需要定义汇编入口 `_start` 并处理多核启动。

### 2.2 内存管理 (Memory) - `interface/mem.rs`

这是移植中最复杂的部分，涉及分页机制和地址空间布局。

#### 必需常量
- `PGSIZE`: 页面大小。
- `VA_MAX`: 虚拟地址空间上限。
- `PHYS_MAP_BASE`: 物理内存线性映射基地址。
- `KERNEL_BASE`: 内核链接基地址。
- `PT_LEVELS`: 页表层级数。
- `PGNUM`: 每级页表的页表项数量。

#### 必需结构体与方法
- `Pte`: 页表项。需实现 `null`, `from`, `as_usize`, `get_ppn`, `set_ppn`, `get_flags`, `set_flags`, `is_valid`。
- `PteFlags`: 标志位。
- `PageTable`: 页表结构，包含 `entries: [Pte; PGNUM]`。
- `perms` 模块: 定义 `VALID`, `READ`, `WRITE`, `EXECUTE`, `USER`, `GLOBAL`, `ACCESSED`, `DIRTY` 等常量。

#### 必需函数
- `flush_tlb(vaddr: Option<VirtAddr>)`: 刷新 TLB。
- `activate_pagetable(root_paddr: PhysAddr)`: 激活页表。
- `deactivate_pagetable()`: 禁用分页。
- `get_vpn_index(va, level) -> VPN`: 获取虚拟地址在某级页表的索引。
- `kpt_setup(kpt: &mut PageTable)`: 设置内核页表（如映射内核段）。

### 2.3 异常与陷阱 (Trap) - `interface/trap.rs`

#### 结构体
- `TrapContext`: 保存寄存器状态。
    - `set_return_value(value)`
    - `get_syscall_args() -> (usize, usize)`
    - `from_trapframe(tf)`
- `TrapFrame`: 陷阱帧。
    - `set_badge(badge)`
    - `get_epc()`, `set_epc(epc)`
    - `set_tf(addr)`
    - `update_context(ctx)`
    - `configure_kernel(...)`: 配置内核态返回信息。

#### 函数
- `vector_init()`: 初始化异常向量表。
- `get_cause() -> TrapCause`: 获取异常原因。
- `get_pc() -> usize`: 获取异常发生时的 PC。
- `advance_pc(pc, offset)`: 移动 PC。
- `get_address() -> VirtAddr`: 获取故障地址（如 PageFault 地址）。
- `get_status() -> usize`: 获取状态寄存器。

### 2.4 中断管理 (IRQ) - `interface/irq.rs`

#### CPU 本地状态
- `enable()`, `disable()`: 开关中断。
- `is_enabled()`: 查询状态。
- `wfi()`: 等待中断。

#### 平台控制器
- `MAX_IRQS`: 最大中断数。
- `init()`: 控制器初始化。
- `init_cpu(cpuid)`: 当前核初始化。
- `mask(irq, cpuid)`, `unmask(irq, cpuid)`: 屏蔽/解除屏蔽。
- `claim(cpuid) -> Option<u32>`: 获取待处理中断号。
- `complete(irq, cpuid)`: 发送 EOI。
- `set_affinity`, `set_priority`: 设置亲和性和优先级。
- `clear_soft()`: 清除软件中断。

### 2.5 CPU 核心能力 - `interface/cpu.rs`

- `MAX_CPUS`: 支持的最大核数。
- `cpu_id()`: 获取当前核 ID。
- `read_cycle()`, `read_time()`: 读取计数器。
- `timer_set_next(next)`: 设置时钟中断。

### 2.6 进程上下文 (Process) - `interface/proc.rs`

- `ProcContext`: 线程上下文。
    - `new()`
    - `configure(entry, stack_top)`
    - `set_fp(fp)`
- `switch_context(old, new)`: 上下文切换汇编实现。

### 2.7 平台功能 (Platform) - `interface/platform.rs`

- `PlatformInfo`: 平台信息结构。
- `init(dtb)`: 平台初始化。
- `bootargs()`: 获取启动参数。
- `shutdown()`, `reboot()`: 关机重启。
- `send_ipi(mask)`: 发送核间中断。
- `memory_range()`, `range()`: 获取内存范围。
- `initrd()`: 获取 InitRD 范围。
- `bootstrap_cpus()`: 启动从核。

### 2.8 控制台 (Console) - `interface/console.rs`

- `init()`: 初始化。
- `print(args)`: 打印格式化字符串。

### 2.9 运行时 (Runtime) - `interface/runtime.rs`

- `backtrace()`: 打印调用栈。

## 3. 库接口实现清单 (`lib/libglenda-rs`)

在 `lib/libglenda-rs/src/arch/<arch>.rs` 中实现 `lib/libglenda-rs/src/arch/api.rs` 定义的接口，并在 `lib/libglenda-rs/src/arch/mod.rs` 中导出。

#### 必需函数
- `unsafe fn syscall(cptr: usize, method: usize) -> usize`: 执行系统调用。
- `unsafe fn syscall_recv(cptr: usize, method: usize) -> (usize, usize)`: 执行接收系统调用 (返回两个值)。
- `unsafe fn panic_break()`: 由于 panic 导致的停机或断点。
- `fn backtrace()`: 用户态回溯。

## 4. 移植步骤建议

1.  **建立骨架**：创建目录结构，使用 `unimplemented!()` 填充所有接口。
2.  **启动代码**：编写启动汇编和 `boot.rs`，实现 `console` 输出。
3.  **内存初始化**：实现 `mem` 模块，配置页表。
4.  **异常与中断**：实现 `trap`, `irq` 模块。
5.  **进程切换**：实现 `proc` 模块的上下文切换。
6.  **平台功能**：完善 `platform` 模块以支持多核和设备树解析。
7.  **库支持**：实现 `libglenda-rs` 中的 syscall 汇编。
