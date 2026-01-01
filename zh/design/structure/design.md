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
        Factotum[Factotum (进程/异常管理)]
        Gopher[Gopher (命名空间/VFS)]
        Unicorn[Unicorn (驱动管理)]
        Tux[Tux (POSIX 服务)]
        App[用户应用]
    end
    subgraph Kernel Space
        Kernel[Glenda 微内核]
    end

    9Ball --> Kernel
    Factotum --> Kernel
    Gopher --> Kernel
    Unicorn --> Kernel
    App --> Factotum
    App --> Gopher
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
    *   负责启动和引导其他核心系统服务（Factotum, Gopher, Unicorn 等）。
    *   向相应的服务分配资源。

### 3.2 Factotum (异常与任务管理器)
*   **角色**：系统的“管家”，负责进程和线程管理。
*   **职责**：
    *   **异常处理**：注册为所有普通进程的 Fault Handler。当发生缺页异常或非法操作时，内核发送消息给 Factotum 处理。
    *   **进程管理**：负责进程的创建 (Spawn)、销毁和生命周期管理。
    *   **内存管理**：维护进程的地址空间布局，处理缺页异常，实现写时复制 (COW) 等策略。

### 3.3 Gopher (命名空间与 VFS 服务器)
*   **角色**：资源管理器，实现类似于 Plan 9 的命名空间。
*   **职责**：
    *   **统一命名空间**：将所有资源（文件、设备、网络连接、进程信息）抽象为文件系统树。
    *   **9P2000 服务器**：基于 IPC 实现完整的 9P2000 协议。所有资源访问都通过标准的 9P 消息 (Tattach, Twalk, Topen, Tread, Twrite) 进行。
    *   **协议转换**：充当 9P 多路复用器/代理。它将 9P 请求转发给特定的资源提供者（驱动程序、文件系统）或为虚拟文件合成响应。

### 3.4 Unicorn (设备驱动管理器)
*   **角色**：驱动管理器。
*   **职责**：
    *   管理硬件设备驱动程序。
    *   将硬件中断 (IRQ) 转换为 IPC 消息并发送给驱动线程。
    *   向 Gopher 注册设备文件。

### 3.5 Tux (POSIX 服务器)
*   **角色**：POSIX 兼容层。
*   **职责**：
    *   为普通应用程序提供标准的 POSIX 系统调用接口（通过 LibC 转发）。
    *   将 POSIX 请求转换为 Glenda 的原生 IPC 调用，与 Factotum 和 Gopher 交互。

### 3.6 Rio (显示管理器)
*   **角色**：图形显示管理器。
*   **职责**：
    *   管理图形硬件 (GPU/Framebuffer)。
    *   提供窗口系统和输入事件分发。

### 3.7 Shell (命令行接口)
*   **角色**：系统交互的主要用户界面。
*   **职责**：
    *   **命令解析**：解释用户命令和脚本。
    *   **进程控制**：通过 Factotum 启动应用程序 (Spawn)。
    *   **文件管理**：与 Gopher 交互进行文件操作。
    *   **环境**：管理环境变量和工作目录。

## 4. 组件交互模型

Glenda 采用 Client-Server 模型。应用程序作为 Client，通过 IPC 向 Server 发送请求。

*   **系统调用路径**：App -> LibC -> IPC -> Tux/Factotum/Gopher -> Kernel
*   **异常处理路径**：App (Fault) -> Kernel -> IPC -> Factotum -> (Fix/Kill) -> Kernel -> App
