# 内存管理设计

## 1. 概述

在 Glenda 的微内核架构中，内核在内存管理中仅担任机制维护者的角色。基于 RISC-V 64-bit (Sv39/Sv48) 体系结构，它通过能力系统严格控制物理帧与虚拟地址空间的映射。

## 2. 物理内存 (Untyped)

所有物理内存均被抽象为 **Untyped Capabilities**。
*   **启动阶段**：内核初始化时将所有可用 RAM 划分为 `Untyped` 区域，并在 `BootInfo` 中记录其物理地址和长度。
*   **管理**：`Untyped` 也是引用计数的。通过 `Retype` 派生的子能力（如 `Frame`）会持有对父 `Untyped` 的引用。
*   **重类型化**：使用 `kernel/src/mem/untyped.rs` 中的逻辑。内核通过维护 `watermark`（水位线）来管理 `Untyped` 内部的线性分配。

## 3. 虚拟内存 (VSpace)

*   **页表结构**：遵循 RISC-V 标准。Sv39 使用三级页表，Sv48 使用四级页表。内核代码在 `kernel/src/mem/pagetable.rs` 中实现 `walk` 和 `map` 逻辑。
*   **地址空间隔离**：每个 TCB 关联一个 `VSpace` 能力。切换任务时，内核更新 `satp` 寄存器并执行 `sfence.vma`。
*   **ASID 管理**：内核在 `kernel/src/proc/asid.rs` 中管理 ASID (Address Space ID)，以优化 TLB 刷新频率。
*   **W^X 强制**：页表映射时强制执行“不可同时写且执行” (W^X) 安全检查。

## 4. 生命周期与回收

Glenda 使用 **引用计数** 确保内存安全：
*   **Capability 自动管理**：内核中的 `Capability` 结构体持有对象指针，`Drop` 时触发 `dec_ref`。
*   **资源回收**：当 `Frame` 或 `PageTable` 的引用计数降至 0 时，其占用的物理内存实际上并未立即归还给 `Untyped` 的 `watermark`（因为这是线性分配的），但该槽位标记为可用，通常通过 `Revoke` 机制进行大规模回收。

## 5. 内核内存布局

*   **HHDM (Direct Mapping)**：内核将所有物理内存直接映射到高虚拟地址（HHDM 偏移量）。
*   **内核镜像**：映射在 `-2GB` 开始的高地址空间。
*   **零堆分配**：内核运行时不使用 `alloc`。所有内核对象（TCB/CNode等）所需的内存均由用户通过 `Retype` 提供预分配的物理空间，内核仅负责初始化。
