# Tux 设计文档

## 1. 简介
Tux 是 Glenda 的 **Linux 兼容环境**。它的架构采用了经典的 **L4Linux** (单一服务器) 模式。Tux 本质上是一个经过移植的 Linux 内核，它作为 Glenda 的一个普通**用户态服务**运行，而不是在一个全虚拟化的虚拟机中。

这种架构允许 Linux 应用程序以接近原生的性能与 Glenda 原生服务并存。

## 2. 核心架构 (L4Linux 模式)

### 2.1 单一服务器 (Single Server)
Tux 服务本身是一个运行在微内核之上的 Linux 内核实例。
*   **Linux 线程**: 所有的 Linux 内核线程实际上只作为 Glenda 上的一个或多个线程运行。
*   **用户空间**: Linux 应用程序（如 `bash`, `gcc`）作为独立的 Glenda 任务（拥有独立的地址空间）运行。

### 2.2 半虚拟化机制 (Paravirtualization Mechanism)
Tux 通过修改 Linux 内核源代码（针对 Glenda 架构移植），将底层敏感指令和硬件访问操作替换为 **Hypercalls (HCall)** 或 Glenda IPC 调用。
1.  **指令替换**: 特权指令（如 `sret`, `sfence.vma`, CSR 读写）被替换为对 Glenda 微内核或 Tux Server 的主动调用。
2.  **HCall 接口**: 使用 `ecall` (作为 HCall) 直接请求 Gluenda 服务，而不是等待异常捕获。这减少了上下文切换的开销。
3.  **中断模拟**: 硬件中断以 IPC 消息的形式投递给 Tux，Tux 在其事件循环中处理这些逻辑中断。

## 3. 配套服务支持 (Supporting Services)

Tux 依赖一套 **Glenda 原生服务** 来运行，有效地将微内核环境视为其“硬件”平台。

### 3.1 VMM 服务 (Chimera/Warren)
*   **角色**: 充当 Tux 的引导加载程序和监视器。
*   **功能**:
    *   将 L4Linux 二进制文件 (`vmlinux`) 加载到内存中。
    *   准备 **BootInfo** 页（包含内存映射、命令行参数）。
    *   提供 **虚拟设备树 (DTB)**，将可用的 IPC 服务描述为伪设备。

### 3.2 设备管理器 (Unicorn)
*   **角色**: Tux 的“南桥”。
*   **功能**:
    *   **中断转发**: Unicorn 接收物理 IRQ 并将其作为异步通知 (Signal) 转发给 Tux。
    *   **MMIO 直通**: 对于性能关键型设备，Unicorn 可以将物理设备寄存器直接映射到 Tux 的地址空间。

### 3.3 I/O 后端
Tux 使用半虚拟化驱动程序 (`virtio-glenda`) 与 I/O 服务通信。

*   **块存储**: 连接到 **Fossil**。Tux 看到一个块设备；Fossil 处理缓存和磁盘驱动程序。
*   **网络**: 连接到 **Gopher**。Tux 看到一个标准的 eth0；Gopher 处理协议卸载或原始数据包传递。
*   **控制台**: 连接到 **9Ball** 以进行日志记录和 TTY 访问。

## 4. 管理与互通 (Management & Interoperability)

Tux 提供了丰富的接口以便与 Glenda 原生环境交互：

*   **控制平面**: Tux 导出一个文件系统接口，暴露 Linux 内部状态（如 `/proc`, `/sys` 的对应物），允许 Glenda 工具 (`rc`, `ls`) 查看 Linux 进程。
*   **数据平面 (共享内存)**: 通过高效的共享内存机制与 **Fossil** (FS) 和 **Gopher** (Net) 交换大数据块。
*   **信号与事件**: 定义了一套机制，使得 Glenda 任务可以向 Linux 进程发送信号，反之亦然。

## 5. 与 Chimera 的关系

虽然 Tux 采用了 L4Linux 架构，Chimera (虚拟化管理器) 仍然扮演辅助角色：
*   如果 Tux 需要运行在硬件虚拟化容器中（以支持未修改的特权指令），Chimera 可作为其底层 VMM。
*   在纯半虚拟化 (Para-virtualized) 模式下，Tux 直接运行在微内核上，不需要 Chimera 介入。默认配置倾向于纯半虚拟化以获得更低的开销。
