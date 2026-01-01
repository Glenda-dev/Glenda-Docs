# Capability 系统 (Cap) 设计

## 1. 理念：对象-Capability 模型

Glenda 使用基于 Capability 的访问控制模型。
*   **无全局 ID**：默认情况下没有可访问的全局 PID 或文件描述符。
*   **显式授权**：线程只有在拥有目标对象的特定 **Capability (Cap)** 时才能执行操作。
*   **粒度**：访问权限是细粒度的（例如，只读、只写、仅授予）。

## 2. CSpace (Capability 空间)

每个进程（或保护域）都有一个 **CSpace**。
*   CSpace 是一个（概念上的）表，将整数索引映射到内核 Capabilities。
*   **CPTR (Capability 指针)**：用户空间代码用于引用 capability 的整数索引（类似于文件描述符 `fd`）。
*   内核管理 CSpace；用户空间无法伪造 capabilities。

## 3. 内核对象

Capabilities 指向特定的内核对象：

| 对象类型 | 描述 |
| :--- | :--- |
| **Untyped** | 所有物理内存的根。表示一块连续的空闲内存。可以“重类型化”为特定的内核对象。 |
| **TCB** | 线程控制块。表示一个线程。调用它允许挂起/恢复/配置线程。 |
| **Endpoint** | IPC 端口。用于消息传递。 |
| **Frame** | 一个物理内存页（例如 4KB）。可以映射到 VSpace 中。 |
| **PageTable** | 硬件页表结构的一个级别。 |
| **IrqHandler** | 管理特定硬件中断的权限。 |
| **CNode** | CSpace 结构中的一个节点（用于存储 caps）。 |
| **Reply** | 一个特殊的、一次性的 capability，用于回复 `Call` 调用。 |

## 4. Capability 操作

### 4.1 调用 (Invocation)
Cap 的主要用途是通过 `sys_invoke` 进行 **调用**。
*   可用的具体方法取决于对象类型。
*   有关每种对象类型的方法的完整列表，请参阅 [系统调用接口](syscall.md)。

### 4.2 重类型化 (对象创建)
**Retype** 操作是创建新内核对象的唯一机制。它将 `Untyped` 内存区域的一部分转换为一个或多个特定的内核对象。

*   **机制**:
    1.  用户在 **Untyped** capability 上调用 `Retype` 方法。
    2.  内核验证请求的内存范围是否在 `Untyped` 区域内且当前空闲。
    3.  内核划分内存，初始化新对象（例如，将 PageTable 清零），并创建相应的 capabilities。
    4.  新的 capabilities 被插入到用户的 CSpace 中。
*   **派生**：新对象被视为 **Capability 派生树 (CDT)** 中原始 `Untyped` capability 的“子对象”。

### 4.3 派生 (Minting)
线程可以从现有的 capability 创建一个新的 capability，可能具有减少的权限。
*   **Mint**: 创建具有特定 **Badge** 或减少权限的副本（例如，读写帧 -> 只读帧）。
*   这是通过在 **CNode** capability 上调用 `Mint` 方法来执行的。

### 4.4 传输 (委托)
Capabilities 可以通过 IPC 消息在 CSpace 之间传输。这是权限委托的主要机制。

*   **机制**:
    1.  **发送者**:
        *   在 `MsgTag` 中设置 `HasCap` 标志。
        *   将要传输的 capability 的 CPTR 放入 `utcb.cap_transfer`。
        *   必须拥有被传输 capability 的 **`Grant`** 权限。
    2.  **接收者**: 在其自己的 `utcb.recv_window` 中指定一个“接收窗口”（指向槽的 CPTR），传入的 capability 应存储在该位置。
    3.  **内核**:
        *   在 IPC 会合期间，内核在发送者的 CSpace 中查找 capability。
        *   它验证 capability 上的 `Grant` 权限。
        *   它将 capability 插入到接收者 CSpace 的指定槽中。
*   **Grant**: 标准操作是复制 capability。发送者保留访问权限，除非他们显式删除它。
*   **用例**: 服务器通过发送 `Frame` cap 授予客户端访问共享内存缓冲区的权限。

### 4.5 撤销与资源回收
Glenda 通过结合 **引用计数** 和 **Capability 派生树 (CDT)** 来确保内存安全和资源核算。

*   **引用计数**: 每个内核对象（TCB, CNode, Endpoint 等）都有引用计数。
    *   **Capability 所有权**: 在 Rust 内核中，`Capability` 结构实现了 `Clone` 和 `Drop`。
    *   **增加**: 克隆 `Capability`（通过 `Clone` 或传递给 `insert` 等方法时）会增加底层对象的引用计数。
    *   **减少**: Drop `Capability`（通过 `Drop` 或覆盖 CNode 槽时）会减少计数。
    *   **销毁**: 当计数达到零时，对象被销毁。如果对象是从 `Untyped` 内存创建的，则该内存逻辑上返回给父 `Untyped` 区域或标记为空闲以供将来重类型化。
*   **Capability 派生树 (CDT)**: CDT 跟踪 capabilities 之间的“父子”关系。
    *   当 `Untyped` 区域被重类型化时，生成的对象是该 `Untyped` capability 的子对象。
    *   当 capability 被 `Minted` 时，新 capability 是原始 capability 的子对象。
*   **Revoke (撤销)**: `Revoke` 操作允许用户递归删除从目标 capability 派生的所有 capabilities。
    *   如果在 `Untyped` capability 上调用 `Revoke`，则从该内存创建的所有对象都将被销毁，并且内存被回收。
    *   这确保了资源提供者始终可以收回其委托给子进程的资源。

## 5. Badges (徽章)

徽章是服务器实现的关键特性。
*   服务器持有 Endpoint 的“主”接收 Cap。
*   服务器为每个客户端“Mint”一个带有唯一 **Badge**（整数标签）的发送 Cap。
*   **不可变身份**: 一旦 capability 被标记，其徽章就无法修改。这确保了客户端无法通过重新标记其收到的 capability 来伪造其身份。
*   **自动传递**: 当线程使用带徽章的 capability 发送消息时，内核会自动提取徽章并将其传递给接收者（通常通过指定的寄存器如 `t0`）。
*   **结果**: 服务器确切地知道哪个客户端发送了消息，而无需内核具有全局“客户端 ID”概念。徽章 *就是* 身份。
