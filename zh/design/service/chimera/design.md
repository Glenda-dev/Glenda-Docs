# Chimera 设计文档

## 1. 简介
Chimera 是 Glenda 的 **虚拟化管理器 (VMM)**。它抽象了硬件虚拟化扩展（如 RISC-V H-Extension 或 Intel VT-x），为 Guest OS 内核、半虚拟化系统和硬件隔离容器提供安全执行环境。

## 2. 职责

*   **HVM 辅助**: 支持硬件虚拟机。
*   **资源分区**: 分配第二阶段页表 (Stage-2 Page Tables)，即客户机物理地址 (GPA) 到宿主机物理地址 (HPA) 的转换。
*   **vCPU 调度**: 将 Guest vCPU 映射到由 Warren 管理的 Glenda 线程。
*   **设备模拟**: 可选地提供设备模型（通常委托给用户态后端驱动程序，如 Tux 或 Unicorn）。

## 3. 架构

Chimera 作为高特权服务（或驱动组件）运行。

### 3.1 虚拟化扩展管理
Chimera 是唯一允许调用虚拟化特定指令（如 `hfence`, `hlv`）的组件。
*   **VMID 管理**: 分配硬件 VMID。
*   **中断注入**: 向 Guest 注入虚拟中断。

### 3.2 运行模式

#### A. 全虚拟化 (HVM)
运行未修改的 Guest OS（例如 Windows, 标准 Linux）。
*   **硬件模拟**: 依赖 **Unicorn** 模拟完整的硬件设备集 (UART, Block, Net)。
*   **隔离**: 使用第二阶段页表进行严格隔离。

#### B. 半虚拟化 (PV)
运行感知 Glenda 环境的修改版 Guest OS（例如专用 RTOS 或 HVM 模式下的 Tux）。
*   **Hypercalls**: Guest 使用 `hVCALL` 直接请求 Chimera/Glenda 服务。
*   **VirtIO**: 使用基于 MMIO 的 VirtIO 进行高效 I/O。

#### C. 容器化 (MicroVM)
为标准 Glenda/Linux 进程提供硬件强制的隔离（类似于 Kata Containers）。
*   **轻量级**: 极小的内存占用，共享只读页面。
*   **文件系统直通**: 使用 `virtio-fs` 直接映射宿主目录。
*   **快速启动**: 针对毫秒级启动进行优化，专门运行单个负载。

## 4. 接口

### 4.1 管理接口 (IPC)
*   `create_vm(config) -> Cap`: 创建新的 VM 实例。
*   `map_memory(vm_cap, gpa, hpa, size, perms)`: 配置第二阶段映射。
*   `create_vcpu(vm_cap) -> ThreadCap`: 创建 vCPU 线程。

### 4.2 Trap 转发协议
当 Guest 触发 Chimera 无法在内核中处理的 VMExit（例如设备 IO）时，Chimera 会向注册的处理端点（通常是 **Unicorn** 或特定的设备模型服务）发送 **Fault IPC**。
