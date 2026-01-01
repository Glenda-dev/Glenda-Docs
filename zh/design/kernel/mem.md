# 内存管理设计

## 1. 概述

在 Glenda 的微内核架构中，内核在内存管理中的角色被简化为 **仅机制**：它确保安全和隔离，但不规定策略。所有物理内存分配和虚拟内存映射策略都在用户空间实现。

## 2. 物理内存 (Untyped)

在 Glenda 中，所有物理内存都通过 **Capability 系统** 进行管理。用户空间无法访问传统的页面分配器（如伙伴系统）。相反，内存表示为 **Untyped Capabilities**。

*   **启动**: 内核从设备树 (DTB) 或 multiboot 信息中识别所有可用的物理内存区域，并将它们包装成 `Untyped` capabilities。
*   **Root Task**: 启动时，Root Task 被授予所有 `Untyped` 区域的 capabilities。它充当系统的主要内存管理器。
*   **Retype 操作**: 这是创建新内核对象的唯一方法。
    *   用户在 `Untyped` capability 上调用 `Retype`。
    *   内核从 `Untyped` 区域划分出一块连续的内存块。
    *   该块被转换为特定类型（例如 `TCB`, `CNode`, `Frame`）。
    *   创建的对象的 capability 被返回给用户。
*   **内存统一**: 内部 `PhysFrame` 抽象与 capability 系统统一。一旦帧被分配给 capability，其生命周期由该 capability 的引用计数管理。

## 3. 生命周期与回收

Glenda 使用 **引用计数** 结合 **Capability 派生树 (CDT)** 来管理内存安全。

*   **引用计数**: 每个内核对象（CNode, TCB, Endpoint 等）都有一个引用计数。
    *   `Capability::clone()` 增加计数。
    *   `Capability::drop()` 减少计数。
*   **回收**: 当 capability 的引用计数达到零时：
    *   内核触发对象的析构函数。
    *   如果对象是容器（如 `CNode`），它会递归 drop 它持有的所有 capabilities。
    *   底层物理内存返回给 `Untyped` 池或在物理内存管理器中标记为空闲。
*   **撤销**: 通过删除 CDT 中的父 capability，内核可以递归地使所有派生的子 capabilities 无效，确保资源所有者可以强制回收内存。

## 3. 虚拟内存 (地址空间)

*   **VSpace**: 地址空间由顶级 `PageTable` capability 表示。
*   **UTCB 映射**: 每个线程都有一个映射到其 VSpace 的 UTCB 页面。此映射在线程创建/配置期间建立。
*   **映射操作**:
    *   通过在 `PageTable` capability 上调用 `sys_invoke` 执行。
    *   **参数**: `FrameCap`, `vaddr`, `permissions`。
    *   内核只是将 PTE (页表项) 插入硬件页表。它不跟踪 VMA (虚拟内存区域) 或管理区域；这是用户空间 pager 的责任。
*   **取消映射操作**:
    *   通过在 `PageTable` capability 上调用 `sys_invoke` 执行。
    *   **参数**: `vaddr`。
    *   移除映射。

## 5. 内核内存

*   **静态内存**: 内核映像和全局数据结构（如 `IRQ_TABLE` 和 `READY_QUEUES`）占用内存的固定部分。内核不使用动态堆分配器（`Vec`, `Box` 等）进行内部操作。
*   **对象存储**: 一旦 Root Task 运行，内核 **从不** 为自己分配内存。进程所需的每个 TCB、页表或 CNode 必须由该进程（或其父进程）通过 `Retype` 机制提供。
    *   内核使用支持 `Untyped` capability 的物理内存来存储对象的元数据。
    *   侵入式数据结构（例如嵌入在 TCB 中的链表）用于管理队列，避免了对辅助内存分配的需求。
*   **严格核算**: 这种设计确保恶意进程无法耗尽内核内存。如果进程用完了自己的 `Untyped` 配额，它就无法创建更多的内核对象。

## 6. Capability 接口

`PageTable` 内核对象通过 `sys_invoke` 暴露以下方法：

| 方法 | 描述 |
| :--- | :--- |
| `Map` | 将 `Frame` capability 映射到虚拟地址。 |
| `Unmap` | 取消映射虚拟地址。 |

`Untyped` 内核对象暴露：

| 方法 | 描述 |
| :--- | :--- |
| `Retype` | 从此区域创建新的内核对象（TCB, CNode, Frame 等）。 |

`CNode` 内核对象（表示 CSpace）暴露：

| 方法 | 描述 |
| :--- | :--- |
| `Copy` | 创建指向同一对象的新 capability（增加引用计数）。 |
| `Mint` | 创建具有减少权限或新 Badge 的新 capability。 |
| `Revoke` | 递归删除从目标派生的所有 capabilities。 |
| `Delete` | 从槽中移除 capability（减少引用计数）。 |
