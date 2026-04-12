# Warren 服务设计文档

## 简介
Warren 是 Glenda 操作系统的 **根任务 (Root Task)**。它是内核启动后第一个被加载的用户空间进程，拥有最高特权。

## 核心职责

1.  **资源抽象与管理**：
    *   接收内核直接传入的 `BootInfo` (位于 `BOOTINFO_VA`)。
    *   管理所有 `Untyped` 物理内存，并提供 `ALLOC` / `FREE` 接口。
    *   管理系统范围内的 `CSpaceManager` 和 `VSpaceManager`。

2.  **进程管理 (Process Management)**：
    *   提供 `CREATE/SPAWN/EXIT/KILL/THREAD_CREATE/GET_CNODE` 等接口。
    *   负责创建进程核心对象（`TCB/CNode/VSpace/UTCB/TrapFrame/KStack`）并写入进程表。
    *   通过 `exit_wrapper` 统一回收进程资源（线程、VSpace 影子状态、Arena 能力槽位）。

3.  **资源服务与能力注册中心**：
    *   提供 `RESOURCE_PROTO` 的 `ALLOC/FREE/DMA_ALLOC/SBRK/GET_CAP/REGISTER_CAP/GET_CONFIG/GET_STATUS`。
    *   维护系统服务 endpoint 注册表并按资源类型分发能力。

4.  **启动与系统服务拉起**：
    *   在 `init()` 中注册核心 endpoint，并通过 `spawn(INIT_NAME)` 拉起 init 服务。

## IPC 协议实现

Warren 实现了以下关键协议：
*   **PROCESS_PROTO (`0x0200`)**：`CREATE`, `SPAWN`, `EXIT`, `KILL`, `THREAD_CREATE`, `GET_CNODE`。
*   **RESOURCE_PROTO (`0x0300`)**：`ALLOC`, `FREE`, `DMA_ALLOC`, `SBRK`, `GET_CAP`, `REGISTER_CAP`, `GET_CONFIG`, `GET_STATUS`。
*   **KERNEL_PROTO (`0x0100`)**：`SYSCALL`, `PAGE_FAULT`, `ILLEGAL_INSTRUCTION`, `BREAKPOINT`, `ACCESS_FAULT`, `ACCESS_MISALIGNED`, `VIRT_EXIT`。

## 与 APE 的职责边界

- **Warren 负责对象所有权与最终回收**：进程/线程内核对象生命周期、全局资源能力分配。
- **APE 负责 Linux ABI 语义**：pid/fd/路径与用户态兼容策略。

因此，对 APE 托管进程，APE 会调用 Warren 的 `create/kill` 并维护本地映射；Warren 仍是底层对象所有者。

## 异常处理说明

并非所有进程故障都必须由 Warren 处理，取决于目标进程的 fault handler 指向：

- 指向 Warren：由 Warren 处理 `KERNEL_PROTO` 异常消息；
- 指向 APE：由 APE 处理 Linux ABI 兼容相关故障路径。
