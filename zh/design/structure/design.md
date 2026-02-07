# Glenda 系统架构设计

Glenda 是一个基于 Capability 的微内核操作系统。其设计理念遵循微内核的最小化原则：内核仅提供最基本的机制，而将策略和大多数系统服务移至用户空间。

## 1. 整体架构

Glenda 的系统架构分为两层：
1.  **内核空间**：Glenda 微内核
2.  **用户空间**：系统服务和应用程序

\`\`\`mermaid
graph TD
    subgraph User Space
        9Ball[9Ball (Root Task)]
        Warren[Warren (进程/异常管理)]
        9P[9P (命名空间服务器)]
        Gopher[Gopher (网络栈)]
        Fossil[Fossil (文件系统)]
        Unicorn[Unicorn (驱动管理)]
        Chimera[Chimera (虚拟化)]
        Tux[Tux (L4Linux 服务)]
        App[用户应用]
    end
    subgraph Kernel Space
        Kernel[Glenda 微内核]
    end

    9Ball --> Kernel
    Warren --> Kernel
    9P --> Kernel
    Gopher --> Kernel
    Fossil --> Kernel
    Unicorn --> Kernel
    Chimera --> Kernel
    Tux --> Kernel
    App --> Warren
    App --> 9P
    App --> Gopher
    App --> Fossil
    App --> Chimera
    App --> Tux
\`\`\`

## 2. 内核层 (Microkernel)

内核主要负责管理最基本的硬件资源并提供受控的访问机制。

### 核心对象
*   **TCB (Thread Control Block)**：线程执行上下文。
*   **Endpoint**：IPC 通信端点，用于线程间的消息传递。
*   **CNode (Capability Node)**：存储 Capability 的容器（类似于文件描述符表）。
*   **PageTable / Frame**：内存管理对象。
*   **Untyped**：未类型的物理内存，用于派生其他对象。
*   **Interrupt**：中断管理对象。

### 核心机制
*   **基于 Capability 的安全机制**：所有资源访问必须通过 Capability 进行，实现细粒度的权限控制。
*   **IPC (进程间通信)**：同步和异步的消息传递机制。
*   **抢占式调度**：基于优先级的抢占式调度。

## 3. 用户空间服务组件

Glenda 的功能主要由一组协作的用户空间服务提供。

### 3.1 9Ball (Root Task / 系统引导程序)
*   **角色**：系统的第一个用户空间进程 (PID 1)。
*   **职责**：
    *   接管内核移交的所有剩余系统资源（Untyped 内存、IO、IRQ Caps）。
    *   负责启动和引导其他核心系统服务（Warren, Gopher, Unicorn 等）。
    *   向相应的服务分配资源。

### 3.2 Warren (异常与任务管理器)
*   **角色**：系统的“管家”，负责进程和线程管理。
*   **职责**：
    *   **异常处理**：注册为所有普通进程的 Fault Handler。当发生缺页异常或非法操作时，内核发送消息给 Warren 处理。
    *   **进程管理**：负责进程的创建 (Spawn)、销毁和生命周期管理。
    *   **内存管理**：维护进程的地址空间布局，处理缺页异常，实现写时复制 (COW) 等策略。



### 3.4 Gopher (网络栈)
*   **角色**：网络栈提供者。
*   **职责**：
    *   **网络管理**：管理网络接口和路由。
    *   **TCP/IP 协议栈**：实现 TCP/IP 协议栈。
    *   **管理接口**: 通过 IPC 暴露网络栈管理接口。
    *   **专用 IPC**：使用高性能共享内存 IPC 进行数据传输。

### 3.5 Fossil (文件系统服务器)
*   **角色**：持久化文件系统服务器。
*   **职责**：
    *   **存储管理**：管理磁盘存储。
    *   **文件系统**：实现文件系统逻辑。
    *   **管理接口**：通过 IPC 暴露文件系统结构。
    *   **专用 IPC**：使用专用 IPC 协议进行批量数据传输。

### 3.6 Unicorn (驱动管理)
*   **角色**：驱动管理器。
*   **职责**：
    *   管理硬件设备驱动程序。
    *   将硬件中断 (IRQ) 转换为 IPC 消息。
    *   **管理接口**：通过 IPC 暴露设备文件。

### 3.7 Tux (Linux 兼容层)
*   **角色**：Linux 半虚拟化兼容层。
*   **职责**：
    *   **Linux Guest**：作为轻量级半虚拟化 Linux Guest 运行在 Chimera 之上。
    *   **ABI 兼容**：为未修改的 Linux 二进制文件提供 ABI 兼容性。
    *   **管理接口**：通过 IPC 暴露进程信息/状态。

### 3.8 Rio (显示管理器)
*   **角色**：图形显示管理器。
*   **职责**：
    *   管理图形硬件 (GPU/Framebuffer)。
    *   提供窗口系统和输入事件分发。
    *   **管理接口**：通过 IPC 暴露窗口管理接口。

### 3.9 Shell (命令行接口)
*   **角色**：系统交互的主要用户界面。
*   **职责**：
    *   **命令解析**：解释用户命令和脚本。
    *   **进程控制**：通过 Warren 启动应用程序 (Spawn)。
    *   **文件管理**：与文件系统交互进行文件操作。

### 3.10 APE (ANSI/POSIX Environment)
*   **角色**：系统库支持。
*   **职责**：
    *   **Musl 支撑**：作为 `musl` 的底层库，实现平台适配器。
    *   **直接交互**：直接通过特定的专用 IPC 协议与系统服务（Gopher, Fossil, Tux）交互，在高性能场景下绕过通用的命名空间。
    *   **协议适配**：将各种协议（如块设备 I/O、网络流、Framebuffer）适配为标准 POSIX 调用。
    *   **环境**：管理环境变量和工作目录。

### 3.11 Chimera (虚拟化/半虚拟化/容器化服务器)
*   **角色**：虚拟机/容器监视器 (VMM)。
*   **职责**：
    *   **虚拟化**：提供硬件虚拟化支持 (利用 RISC-V H-Extension 等硬件扩展或软件模拟)。
    *   **资源隔离**：管理 Guest 操作系统或容器的隔离环境。
    *   **Tux 宿主**：作为 **Tux** Linux 兼容层的 Hypervisor/宿主。
    *   **管理接口**：通过 IPC 暴露 VM 管理接口。

## 4. 组件交互模型

Glenda 采用混合的 Client-Server 模型。

*   **数据平面 (Data Plane)**：组件之间使用 **专用 IPC 协议** (例如共享内存环形缓冲区、直接 Capability 调用) 进行高吞吐量和低延迟的操作。
*   **控制/管理平面 (Control/Management Plane)**：组件通过 IPC 暴露 **管理接口**。这为检查、配置和管理系统提供了方式。

*   **系统调用路径**：App -> LibC -> APE -> IPC (专用协议) -> Gopher/Fossil/Chimera -> Kernel
*   **异常处理路径**：App (Fault) -> Kernel -> IPC -> Warren -> (Fix/Kill) -> Kernel -> App
