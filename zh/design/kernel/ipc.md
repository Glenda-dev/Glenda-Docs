# 进程间通信 (IPC) 设计

## 1. 概述

在 Glenda 的微内核架构中，进程间通信 (IPC) 是协调和数据交换的基本机制。与组件直接调用函数的宏内核不同，微内核组件（驱动程序、文件系统、应用程序）运行在隔离的地址空间中，必须使用 IPC 进行交互。

## 2. 设计原则

*   **同步 (会合)**：标准 IPC 是同步的。发送者阻塞直到接收者准备好，反之亦然。在握手期间，数据直接在线程之间复制。
*   **异步通知**：用于硬件中断和简单的信号传递。这些是非阻塞的，只携带一个“Badge”（身份/事件代码），没有数据负载。
*   **短消息零拷贝**：最频繁的消息完全通过 CPU 寄存器传递。
*   **基于 Capability**：IPC 是通过调用 **Endpoints** 的 capabilities 执行的。

## 3. 消息结构

IPC 消息由 **UTCB (用户线程控制块)** 布局定义：

1.  **消息标签 (`msg_tag`)**：一个描述消息协议、标签和消息中字数的单字。
    *   **长度 (Bits 0-3)**：使用的消息寄存器 (`mrs_regs`) 数量 (0 到 7)。
    *   **标志 (Bits 4-15)**：
        *   Bit 4: `HasCap` - 指示正在传输 capability。
        *   Bit 5: `HasBuffer` - 指示 IPC 缓冲区用于大负载。
    *   **标签 (Bits 16-63)**：特定于协议的标识符。例如，`0xFFFF` 保留用于内核生成的 Fault 消息。

2.  **消息寄存器 (`mrs_regs`)**：在 trap 期间直接在 CPU 寄存器（例如 RISC-V 上的 `a1`-`a7`）中传递的 7 个机器字 (MR1-MR7)。
3.  **Capability 传输字段**：
    *   **`cap_transfer`**: 发送者希望传输的 capability 的 CPTR。
    *   **`recv_window`**: 接收者希望存储传入 capability 的 CPTR (CNode + Index)。
4.  **IPC 缓冲区**：用于较大负载的页面对齐缓冲区。如果 UTCB 中的 `ipc_buffer_size` 非零，内核执行从发送者缓冲区到接收者缓冲区的 `memcpy`。
5.  **Badge**: 一个机器字（在 `t0` 中传递），用于标识发送者的 capability 或中断源。内核自动从用于调用 IPC 操作的 capability 中提取徽章。

## 4. 用户线程控制块 (UTCB)

UTCB 是内核和用户空间之间的关键共享内存区域。

*   **Capability 管理**：支持 UTCB 的物理帧作为 TCB 中的 **Frame Capability** 进行管理。这允许使用标准 IPC capability 传输在线程之间委托或共享 UTCB。
*   **固定映射**：每个线程的 UTCB 都映射在其 VSpace 中的固定虚拟地址（例如 `0x8000_0000`）。
*   **生命周期**：UTCB 的生命周期与 TCB 的引用计数绑定。当 TCB 被销毁时，其 UTCB capability 被 drop，可能会回收物理内存。

## 5. IPC 操作

### 5.1 同步发送/接收
*   **`Send` (方法 1)**：阻塞调用者，直到接收者在目标 Endpoint 上等待。
*   **`Recv` (方法 2)**：阻塞调用者，直到发送者到达目标 Endpoint 或有挂起的通知可用。
*   **`Call` (方法 3)**：组合的发送-然后-接收操作，为服务器提供隐式 Reply capability 以进行响应。这用于 RPC 风格的通信。

### 5.2 异步通知
*   **`Notify` (方法 4)**：非阻塞操作，将徽章传递给 Endpoint。如果没有接收者在等待，徽章将按位或运算到 Endpoint 的 `notification_word` 中。

### 5.3 回复
*   **`Reply` (Reply 对象上的方法 1)**：由服务器用于回复执行了 `Call` 的客户端。Reply 对象是内核在 `Call` 期间提供的特殊的、一次性的 capability。

## 6. Endpoints

Endpoints 是 IPC 的会合点。为了避免动态内存分配，它们使用侵入式链表和按位通知字。

*   **`send_queue`**: 指向试图发送到此端点时被阻塞的 TCB 的双向链表的头尾指针。链接直接存储在 TCB 中。
*   **`recv_queue`**: 指向等待此端点上的消息时被阻塞的 TCB 的双向链表的头尾指针。
*   **`notification_word`**: 替换用于异步通知的动态队列的按位字 (usize)。
    *   当发送通知（例如，来自 IRQ）时，发送者的徽章按位或运算到此字中。
    *   这允许多个事件同时挂起而无需分配内存，但将同一事件（同一位）的多次发生折叠为单个通知。

## 7. Capability 委托

消息可以将 Capability 从发送者的 CSpace 传输到接收者的 CSpace。这对于资源管理和安全委托至关重要。

*   **发送者**:
    *   在 `MsgTag` 中设置 `HasCap` 标志。
    *   将要发送的 capability 的 CPTR 放入 `utcb.cap_transfer`。
    *   必须拥有 capability 的 `Grant` 权限。
*   **接收者**:
    *   通过 `utcb.recv_window` 在其 CSpace 中指定目标槽。
*   **内核**:
    *   在会合期间（在 `copy_msg` 中），内核在发送者的 CSpace 中查找 capability。
    *   它验证 `Grant` 权限。
    *   它将 capability 插入到接收者 CSpace 的指定 `recv_window` 中。
    *   根据特定对象类型和派生规则，capability 实际上被“移动”或“复制”。
