# 9Ball 设计文档

## 1. 简介
9Ball 是 Glenda 操作系统的 Root Task（相当于 PID 1）。它是内核启动的第一个用户空间程序。其主要职责是引导用户空间环境，特别是初始化 `Factotum` 服务管理器，然后协调其他基本系统服务的启动。

## 2. 职责

*   **系统引导**: 初始化全局 Capability 空间和内核移交的基本资源。
*   **Factotum 初始化**: 加载并启动 `Factotum` 服务。由于 `Factotum` 是任务管理器，9Ball 必须在将进程创建委托给 Factotum 之前，手动（使用 `libglenda` 辅助函数）执行 Factotum 的初始加载。
*   **服务编排**: 一旦 Factotum 运行起来，9Ball 就充当系统配置管理器，根据配置指示 Factotum 启动其他核心服务（例如 Unicorn, Gopher, Rio）。
*   **系统监控**: 引导完成后，9Ball 进入监督循环，监控关键服务的存活状态并处理系统范围的电源状态（关机/重启）。
*   **日志管理**: 在专用日志守护进程可用之前，捕获并缓冲早期启动服务的日志。

## 3. 启动序列

### 第一阶段：内核移交
内核启动 9Ball 并通过初始 Capability 空间 (CSpace) 授予其所有可用的系统资源（未类型化内存、IRQ Capabilities、设备帧等）。

### 第二阶段：启动 Factotum
1.  9Ball 定位 `Factotum` 二进制文件（通常由引导加载程序作为引导模块提供）。
2.  9Ball 为 Factotum 创建一个新的保护域 (CSpace 和 VSpace)。
3.  **资源委派**: 9Ball 将大部分系统资源（未类型化内存、IRQ 控制权）转移给 Factotum。这使得 Factotum 能够履行其作为系统其余部分的内存和任务管理器的角色。
4.  9Ball 启动 Factotum 线程。

### 第三阶段：启动系统服务
一旦 Factotum 处于活动状态，9Ball 就会与其建立 IPC 通道。然后，9Ball 通过向 Factotum 发送请求来启动其他服务。

**启动服务（例如 Gopher）的工作流程：**
1.  9Ball 读取 initrd 中的服务二进制文件。
2.  9Ball 向 Factotum 发送 `CreateProcess` IPC 消息，提供二进制数据或位置。
3.  Factotum 解析 ELF，分配内存并创建线程。
4.  Factotum 将进程句柄返回给 9Ball。
5.  9Ball 向新进程发送 `Start` 消息。

### 第四阶段：监督循环
启动序列完成后，9Ball 进入阻塞循环，等待 IPC 消息或故障通知。
*   **故障处理**: 如果关键服务崩溃，Factotum（作为异常处理程序）可能会通知 9Ball。
*   **系统电源**: 9Ball 处理关机或重启请求。

## 4. 接口

### 与 Factotum 的交互
9Ball 作为 Factotum 的客户端进行进程管理。

*   **接口**: `org.glenda.proc.Manager` (概念上的)
*   **方法**:
    *   `Spawn(name: str, binary: Blob Address) -> Handle`
    *   `Kill(handle: Handle)`
    *   `GetStatus(handle: Handle) -> Status`

## 5. 关键服务列表
9Ball 负责确保以下服务正在运行：
1.  **Factotum**: 异常与任务管理器（用户空间的“内核”）。
2.  **Unicorn**: 设备驱动管理器。
3.  **Gopher**: VFS 和命名空间服务器。
4.  **Rio**: 窗口和显示服务器（对于无头模式是可选的）。

## 6. 日志管理

9Ball 在启动阶段的系统可观测性方面发挥着至关重要的作用。

### 早期启动日志
在显示服务器 (Rio) 或 POSIX 服务器 (Tux) 处于活动状态之前，9Ball 充当主要的日志接收器。
1.  **内核控制台**: 9Ball 利用内核的调试打印功能（例如 `sys_debug_print`）将关键的启动里程碑输出到串行控制台。
2.  **服务输出捕获**: 当 9Ball 指示 Factotum 启动服务时，它会提供一个 IPC 端点 Capability 或共享内存环形缓冲区，用作该服务的 `stdout`/`stderr`。

### 日志缓冲
9Ball 维护一个内存中的 **环形缓冲区**（例如 64KB）来存储所有受监督服务的日志。
*   这确保了如果服务在启动期间崩溃，其最后的输出得以保留。
*   如果启动失败，可以通过开发人员工具查询此缓冲区或将其转储到屏幕上。

### 日志移交
一旦系统完全启动，9Ball 可以选择停止充当日志接收器，并将新的服务输出重定向到专用的系统记录器（例如在 Tux 上运行的 `syslogd`），或者继续充当内核日志缓冲区。

## 7. 未来考虑
*   **可配置启动**: 支持读取 `init.toml` 配置文件来定义要启动的服务及其参数。
*   **并行启动**: 并行启动独立服务以加快启动速度。
*   **恢复策略**: 为每个服务定义重启策略（总是、失败时、从不）。
