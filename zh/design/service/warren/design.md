# Warren 服务设计文档

## 简介
Warren 是 Glenda 操作系统的 **根任务 (Root Task)**。它是内核启动后第一个被加载的用户空间进程，拥有最高特权。

## 核心职责

1.  **资源抽象与管理**：
    *   接收内核直接传入的 `BootInfo` (位于 `BOOTINFO_VA`)。
    *   管理所有 `Untyped` 物理内存，并提供 `ALLOC` / `FREE` 接口。
    *   管理系统范围内的 `CSpaceManager` 和 `VSpaceManager`。

2.  **进程管理 (Process Management)**：
    *   提供 `SPAWN`, `EXIT` 接口。
    *   负责解析 ELF 文件（通过 `elf.rs`），建立初始映射，分配 `TCB`。
    *   管理 PID 到 `Process` 结构体的映射。

3.  **系统调用模拟与权限代理**：
    *   监听 `MONITOR_CAP` 端点。
    *   处理来自普通进程的 `sbrk`, `DMA_ALLOC`, `GET_CONFIG` 等请求。
    *   作为内核异常（Fault）的处理者。

4.  **服务发现与配置**：
    *   解析 `initrd` (位于 `INITRD_VA`)，启动名为 `init` 的首个二级子服务（通常是 `nineball`）。
    *   提供文件系统 `fossil` 和网络 `gopher` 所需的基础 Capability。

## IPC 协议实现

Warren 实现了以下关键协议：
*   **PROCESS_PROTO (0x1)**：包含 `SPAWN`, `EXIT`, `THREAD_CREATE`, `GET_CNODE` 等。
*   **RESOURCE_PROTO (0x3)**：包含 `ALLOC` (分配能力), `DMA_ALLOC` (物理内存分配), `SBRK` (堆扩展), `GET_CONFIG` (从 initrd 获取文件) 等。
*   **KERNEL_PROTO (0x2)**：用于处理内核转发的 `PAGE_FAULT`, `ILLEGAL_INSTRUCTION` 等异常。

## 故障处理流程
当子进程触发缺页异常时：
1. 内核陷阱捕获异常。
2. 内核将上下文打包为 `KERNEL_PROTO` 消息发送给 Warren。
3. Warren 在 `page_fault` 逻辑中检查触发进程的 VSpace。
4. 若为按需分配或堆扩展，Warren 分配物理帧并 Map 到目标进程，然后回复执行。
5. 若为非法访问，Warren 执行 `kill` 逻辑。
