# Glenda 微内核设计文档

## 1. 架构概览

Glenda 是一个类 L4 的微内核架构。其核心理念是 **极简主义** 和 **机制与策略分离**。

内核运行在 Supervisor 模式 (S-Mode)，仅提供构建操作系统所需的最基本机制。所有更高级别的抽象，包括设备驱动程序、文件系统和网络协议，都作为用户空间服务实现。

## 2. 内核范围

微内核负责以下最小功能集：

*   **进程间通信 (IPC)**：线程间数据交换和同步的基本机制。
*   **地址空间管理**：管理虚拟内存映射和页表。
*   **线程管理**：线程创建、调度和上下文切换。
*   **Capability 系统 (Cap)**：所有内核对象的细粒度访问控制。
*   **中断处理**：将硬件中断转换为 IPC 消息，通过 Capability 系统发送给特定的用户空间处理程序。

### 移至用户空间的组件
*   **设备驱动程序**：UART, VirtIO 等。
*   **文件系统**：VFS, FAT32 等。
*   **系统服务**：进程管理，内存服务器。

## 3. 关键机制

### 3.1 Capability 系统 (CSpace)
安全和资源管理基于 Capabilities。Capability 是一个受内核保护的令牌，指向具有特定访问权限的内核对象。

*   **生命周期管理**：内核对象通过 **引用计数** 进行管理。`Capability` 结构实现了 `Clone`（增加引用）和 `Drop`（减少引用）。当计数达到零时，底层物理内存被回收。
*   **Untyped 内存**：所有物理内存最初都表示为 `Untyped` capabilities。内核对象（TCB, CNode 等）是通过“重类型化 (retyping)” `Untyped` 内存创建的。这将内存管理与 capability 系统统一起来。
*   **Badge (徽章)**：在 `Mint` 操作期间附加到 capability 的 64 位不可变标识符。
    *   **身份**：服务器使用徽章来识别客户端。
    *   **不可变性**：一旦设置了徽章，持有者就无法修改或删除它。
    *   **通知**：对于通知对象，徽章充当位掩码，在发出信号时进行 OR 运算。
*   **对象**：TCB (线程控制块), Endpoint (IPC 端口), Frame (物理页), Untyped (空闲内存), IrqHandler, PageTable, CNode。
*   **CSpace**：每个进程都有一个 Capability 空间 (CSpace) 存储其 capabilities。
*   **调用**：系统调用是通过调用 capability 执行的（例如，`sys_invoke(cptr, args)`）。

### 3.2 进程间通信 (IPC)
IPC 是线程交互的唯一方式。
*   **同步**：IPC 通常是同步的，以避免缓冲开销。
*   **消息传递**：短消息通过寄存器传递；扩展消息和 capabilities 通过 **UTCB** (用户线程控制块) 传递。
*   **Badge 传递**：在 IPC 期间，内核从发送者的 capability 中提取徽章并将其注入接收者的上下文（例如，寄存器 `t0`）。这提供了一个安全的、内核验证的身份。
*   **Endpoints**：IPC 指向 Endpoint 对象，而不是直接指向线程。

### 3.3 异常处理 (应用程序中断)
内核不在内部处理缺页异常或其他应用程序异常。相反，这些被视为 **应用程序中断**。
1.  当线程发生故障（例如，缺页异常）时，内核挂起该线程。
2.  内核将故障详细信息保存到线程的 UTCB。
3.  内核生成描述故障的 IPC 消息。
4.  此消息被发送到线程注册的 **异常处理程序**（存储在 TCB 中的 Endpoint capability）。
5.  处理程序解决故障（例如，分配一个帧）并回复内核。
6.  内核恢复发生故障的线程。

### 3.4 Root Task (根任务)
**Root Task** 是内核启动的第一个用户空间进程。
*   它接收所有可用系统资源（所有空闲内存，所有 IO 端口/IRQ）的 capabilities。
*   **CSpace 布局**：Root Task 具有固定的初始 CSpace 布局：
    *   Slot 0: Root CNode Capability
    *   Slot 1: Root VSpace (PageTable) Capability
    *   Slot 2: Root TCB Capability
    *   Slot 3: Root UTCB Frame Capability
    *   Slot 4+: Untyped 内存 Capabilities
*   它充当初始资源管理器和 IrqHandler。
*   它负责引导其余的用户空间环境（启动驱动程序、FS 服务器、Shell）。

### 3.5 作为 TCB 的内核抽象
内核本身使用 TCB 抽象进行内部管理：
*   **Idle 线程**：每个 CPU 一个最低优先级的 TCB，当没有其他线程就绪时运行。
*   **内核线程**：虽然内核主要是事件驱动的，但在必要时可以支持内核模式线程用于特定的后台任务，尽管主要设计避免使用它们，而倾向于用户空间处理程序。

## 4. 启动过程

1.  **Bootloader**:
    *   **OpenSBI (RISC-V)**: 直接跳转到内核。内核使用嵌入式 payload。
    *   **GRUB (x86/UEFI)**: 加载内核和 `payload.bin`。传递 Multiboot 信息。
2.  **内核初始化**:
    *   初始化 CPU, Trap Vector, 内存分配器。
    *   **Payload 发现**:
        *   检查 Multiboot 信息中的模块。
        *   回退到嵌入式 `_payload_start`。
    *   创建 Root Task。
    *   用所有资源填充 Root Task 的 CSpace。
3.  **切换到用户模式**: 内核跳转到 Root Task 入口点。
4.  **用户空间初始化**:
    *   Root Task 初始化其分配器。
    *   Root Task 启动驱动程序进程 (UART, Timer)。
    *   Root Task 启动文件系统服务器。
    *   Root Task 启动 Shell/Init 进程。

## 5. 系统调用接口

*   **`sys_invoke(cptr, ...)`**: 在内核对象（TCB, PageTable 等）上调用方法。
