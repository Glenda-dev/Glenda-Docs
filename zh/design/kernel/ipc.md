# 进程间通信 (IPC) 设计

## 1. 概述

在 Glenda 的微内核架构中，进程间通信 (IPC) 是协调和数据交换的基本机制。与组件直接调用函数的宏内核不同，微内核组件（驱动程序、文件系统、应用程序）运行在隔离的地址空间中，必须使用 IPC 进行交互。

## 2. 设计原则

*   **同步 (会合)**：标准 IPC 是同步的。发送者阻塞直到接收者准备好，反之亦然。在握手期间，数据直接在线程之间复制。
*   **异步通知**：用于硬件中断和简单的信号传递。这些是非阻塞的，只携带一个“Badge”（身份/事件代码），没有数据负载。
*   **短消息零拷贝**：最频繁的消息完全通过 CPU 寄存器传递。
*   **基于 Capability**：IPC 是通过调用 **Endpoints** 的 capabilities 执行的。

## 3. 消息结构

IPC 消息的内核处理主要由 `kernel/src/ipc/mod.rs` 及其子模块负责。

1.  **消息标签 (`msg_tag`)**：一个描述消息协议、标签和消息中字数的单字。
    *   **Length (Bits 0-3)**：MRS 的有效长度。
    *   **Flags (Bits 4-7)**：`HAS_CAP` (0x10), `HAS_BUFFER` (0x20)。
    *   **Protocol (Bits 8-15)**：协议 ID。
    *   **Label (Bits 16-63)**：操作标签。

2.  **消息寄存器 (`mrs_regs`)**：定义在 `lib/libglenda-rs/src/ipc/utcb.rs`。目前 `MAX_MRS` 为 8。

3.  **Capability 传输**：
    *   `cap_transfer`: 当前持有要发送的能力的 CPTR。
    *   `recv_window`: 接收端存放能力的 CPTR。
    *   `reply_window`: 接收端存放生成的 `Reply` 能力的 CPTR。

4.  **UTCB (User Thread Control Block)**：
    *   每个线程都有一个位于 `UTCB_VA` 的共享区域。
    *   内核通过 `kernel/src/ipc/utcb.rs` 提供对当前 TCB 所关联 UTCB 帧的访问权限。

## 4. 同步 IPC 机制

*   **直接拷贝**：内核在执行 `copy_msg` (见 `kernel/src/ipc/mod.rs`) 时，直接从 Sender 的 UTCB 物理帧拷贝数据到 Receiver 的 UTCB 物理帧，避开了不必要的虚拟地址操作。
*   **优先级继承**：如果高优先级线程向低优先级线程发送 IPC (或 Call)，内核会自动将低优先级线程的优先级临时提升，并传播到其 `ipc_partner` 链上。
*   **时间片捐赠**：`Call` 操作会发生时间片捐赠，服务器使用客户端的时间片运行，直到 `Reply` 为止。

## 5. 异步通知 (Notify)

*   **极简非阻塞**：调用者发送一个 64-bit 的 Badge 字。
*   **合并原理**：Endpoint 维护一个 `notification_word`。多个通知会进行位或合并，避免产生多个队列项。

## 6. 能力委托 (Transfer)

*   内核在拷贝消息的同时，检查 `HAS_CAP` 标志。
*   如果有效，内核通过 `transfer_cap` 函数 (在 `kernel/src/ipc/mod.rs` 定义) 在 CSpace 之间搬运能力副本，并验证 `Grant` 权限。
