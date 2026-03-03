# Glenda 系统架构设计

Glenda 是一个基于 Capability 的微内核操作系统。其设计理念遵循微内核的最小化原则：内核仅提供最基本的机制，而将策略和大多数系统服务移至用户空间。

## 1. 整体架构

Glenda 的系统架构分为两层：
1.  **内核空间**：Glenda 微内核
2.  **用户空间**：系统服务和应用程序

```mermaid
graph TD
    subgraph User Space
        Warren[Warren (Root Task)]
        Nineball[Nineball (Service)]
        Gopher[Gopher]
        Fossil[Fossil]
        Unicorn[Unicorn]
        App[用户应用]
    end
    subgraph Kernel Space
        Kernel[Glenda 微内核]
    end

    Warren --> Kernel
    Nineball --> Kernel
    Gopher --> Kernel
    Fossil --> Kernel
    Unicorn --> Kernel
    App --> Warren
    App --> Gopher
    App --> Fossil
```

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

### 3.1 Warren (Root Task / 监视器)
*   **角色**：系统的第一个用户空间进程，即 Root Task。
*   **职责**：
    *   解析 `BootInfo` 以发现 `initrd` 的位置及物理内存的使用情况。
    *   接管内核移交的所有剩余系统资源（Untyped 内存、IO、IRQ Caps）。
    *   管理全局资源。
    *   负责衍生 (spawn) 其他所有的系统功能服务（如 `nineball`, `unicorn`）。
    *   作为系统的 "Monitor"，通过 `MONITOR_CAP` 接收子系统 IPC 并处理请求。

### 3.2 Nineball / Unicorn (核心特性服务)
*   **角色**：系统构建块。
*   **职责**：
    *   由 Warren 衍生出的一系列独立服务。
    *   共同提供底层硬件或特性的抽象机制。

### 3.3 Gopher (网络栈)
*   **角色**：网络栈提供者。
*   **职责**：
    *   **网络管理**：管理网络接口和路由。
    *   **TCP/IP 协议栈**：实现 TCP/IP 协议栈。

### 3.4 Fossil (文件系统服务器)
*   **角色**：持久化文件系统服务器。
*   **职责**：
    *   **存储管理**：管理磁盘存储。
    *   **文件系统**：实现文件系统逻辑。

### 3.5 APE (ANSI/POSIX Environment)
*   **角色**：系统库支持。
*   **职责**：
    *   **Musl 支撑**：作为 `musl` 的底层库，实现平台适配器。

## 4. 组件交互模型

Glenda 采用混合的 Client-Server 模型。

*   **数据平面 (Data Plane)**：组件之间使用 **专用 IPC 协议**进行高吞吐量和低延迟的操作。
*   **控制/管理平面 (Control/Management Plane)**：组件通过 IPC 暴露 **管理接口**。
*   **服务寻址**：服务主要基于内核提供的强类型端点（Endpoints）进行直接通信，依赖 Warren 建立的能力连接。
*   **系统调用路径**：User App -> `libglenda-rs` -> 高级 OS 系统调用 (通过 `MONITOR_CAP` IPC 转发至 Warren) 或通过专用 Cap 调用。

