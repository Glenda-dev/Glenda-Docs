# Glenda 微内核迁移待办事项列表

## 阶段 0: 构建系统基础设施 (参考: `xtask.md`)
- [x] **依赖**: 添加 `serde` 和 `toml` 到 `xtask/Cargo.toml`。
- [x] **配置**: 创建 `config.toml` 用于定义启动模块。
- [x] **模块化重构**:
    - [x] 将 `xtask` 拆分为模块: `config`, `build`, `pack`, `qemu`, `fs`。
    - [x] 实现 `config.rs` 以解析 `config.toml`。
    - [x] 实现 `pack.rs` 以生成 `target/bootfs/payload.bin` (Header + Modules)。
    - [x] 实现 `build.rs` 以确保 `pack` 在内核构建之前运行。
- [x] **内核集成 (Incbin 策略)**:
    - [x] 使用 `.incbin` 创建 `kernel/src/asm/payload.S` 以包含 `payload.bin`。
    - [x] 更新 `kernel/build.rs` 以监视 `payload.bin` 的更改。
- [ ] **GRUB / Multiboot 支持**:
    - [ ] 添加 `multiboot2` 头到内核汇编入口。
    - [ ] 在 Rust 中实现 Multiboot2 标签解析器。
    - [ ] 抽象 `PayloadSource` trait (Embedded vs Multiboot)。
    - [ ] 实现 `xtask dist` 以生成可启动 ISO。

## 阶段 1: 内核清理与剥离 (参考: `mem.md`, `proc.md`)
- [x] **移除内核模式驱动程序**:
    - [x] 注释掉 `kernel/src/drivers`。
    - [x] 将驱动程序代码移动到 `staging/drivers` 以供稍后移植。
- [x] **移除内核模式文件系统**:
    - [x] 注释掉 `kernel/src/fs`。
    - [x] 将 FS 代码移动到 `staging/fs`。
- [x] **简化内存管理**:
    - [x] 移除 `kernel/src/mem/{mmap.rs, uvm.rs}`。
    - [x] 从内核中移除 VMA (虚拟内存区域) 跟踪。
    - [x] 保留 `pmem.rs` (物理帧分配器) 和 `pagetable.rs`。
- [x] **简化进程管理**:
    - [x] 将 `Process` 结构重命名为 `TCB` (线程控制块)。
    - [x] 移除 `parent`, `exit_code`, `sleep_chan`, `mmap_head`。
    - [x] 移除 `fork`, `exec`, `wait` 逻辑。

## 阶段 2: 核心微内核机制 (参考: `cap.md`, `ipc.md`)
- [x] **Capability 系统 (`kernel/src/cap.rs`)**:
    - [x] 定义 `CapType` 枚举 (`Untyped`, `TCB`, `Endpoint`, `Frame`, `PageTable`, `IrqHandler`, `CNode`, `Reply`)。
    - [x] 定义 `Capability` 结构 (对象引用 + 权限 + 徽章)。
    - [x] 使用基数树或单级表 (`CNode`) 实现 `CSpace` (Capability 空间)。
    - [x] 实现 `Cap::derive` (Mint) 和 `Cap::revoke`。
- [x] **线程控制块 (`kernel/src/proc/tcb.rs`)**:
    - [x] 添加 `cspace_root` (Capability 到根 CNode)。
    - [x] 添加 `vspace_root` (Capability 到根 PageTable)。
    - [x] 添加 `fault_handler` (Capability 到异常处理程序 Endpoint)。
    - [x] 添加 `ipc_buffer` (用户映射的缓冲区，用于长消息)。
    - [x] 添加 `state` 枚举 (`Running`, `Ready`, `BlockedSend`, `BlockedRecv`, `BlockedCall`)。
- [x] **IPC 子系统 (`kernel/src/ipc`)**:
    - [x] 定义 `IPCMessage` 格式 (Tag + 寄存器 + 可选缓冲区)。
    - [x] 实现 `Endpoint` 内核对象 (等待 TCB 的队列)。
    - [x] 实现 `Notification` 内核对象 (通过带有 `notification_word` 的 `Endpoint`)。
    - [x] 实现同步传输逻辑 (直接上下文切换)。

## 阶段 3: 系统调用接口 (参考: `syscall.md`)
- [x] **移除旧系统调用**:
    - [x] 删除 `kernel/src/syscall/{fs,proc,brk,mmap}.rs`。
- [x] **实现微内核系统调用**:
    - [x] `sys_invoke(cptr, ...)`: 通用调用。
- [x] **实现对象调用**:
    - [x] `Untyped::Retype`: 从空闲内存创建新对象。
    - [x] `PageTable::Map/Unmap`: 操作地址空间。
    - [x] `TCB::Configure/Resume`: 设置和启动线程。
    - [x] `CNode::Mint/Copy/Move`: 管理 capabilities。
    - [x] `Reply::Reply`: 回复 Call。

## 阶段 4: 中断与定时器 (完善)
- [ ] **修复 S-Mode 定时器**:
    - [ ] 从 `kernel/src/trap/handler/vector.rs` 中移除 M-Mode CSR 访问 (`mscratch`, `mtimecmp`)。
    - [ ] 使用 SBI 调用 (`sbi_set_timer`) 进行定时器管理。
- [ ] **中断处理**:
    - [ ] 实现 `IrqHandler` capability 调用 (Ack, SetNotification)。
    - [ ] 将硬件 IRQ 映射到 `Notification` 对象。

## 阶段 5: Root Task (9ball) 实现
- [ ] **项目结构**:
    - [ ] 将 `9ball` 初始化为 `no_std` 二进制文件。
    - [ ] 确保它链接到 `libglenda`。
- [ ] **引导**:
    - [ ] 在用户空间实现 `BootInfo` 解析 (匹配内核布局)。
    - [ ] 实现基本的物理内存分配器 (管理 `Untyped` caps)。
    - [ ] 实现基本的 CSpace 管理器 (跟踪槽)。

## 阶段 6: 用户空间基础 (libglenda)
- [x] **运行时支持**:
    - [x] 实现 `crt0` (入口点, 栈设置)。
    - [x] 为用户空间实现 `panic_handler`。

## 阶段 7: 基础服务
- [ ] **串口驱动**:
    - [ ] 用户空间驱动程序监听来自 UART IRQ 的 Notification。
- [ ] **定时器驱动**:
    - [ ] 管理定时器事件的用户空间驱动程序。
- [ ] **图形驱动**:
    - [ ] 用于帧缓冲区访问的用户空间驱动程序。

## 阶段 8: POSIX & Shell
- [ ] **Tux Server**:
    - [ ] 基本 POSIX 信号处理。
    - [ ] 通过 TCB 操作模拟 `fork`/`exec`。
- [ ] **Shell**:
    - [ ] 基本命令行界面。

## 阶段 9: SMP 支持
- [ ] **多 Hart 启动**:
    - [ ] 扩展启动过程以通过 SBI 启动辅助 harts。
- [ ] **处理器间中断 (IPI)**:
    - [ ] 实现通过 SBI 调用发送 IPI。

## 阶段 10: 网络与文件系统
- [ ] **网络栈**:
    - [ ] 用户空间网络驱动程序。
- [ ] **文件系统**:
    - [ ] 用户空间文件系统驱动程序。

## 阶段 11: 图形用户界面
- [ ] **Wayland Compositor**:
    - [ ] 基本窗口管理。
