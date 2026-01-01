# Factotum 设计文档

## 1. 简介
Factotum 是 Glenda 微内核系统的中央资源和任务管理器。虽然内核提供了机制（capabilities, 线程, 地址空间），但 Factotum 提供了 *策略* 和高级管理。它充当用户空间的“内核”。

## 2. 职责

*   **内存管理**:
    *   从 9Ball 接收大部分系统内存 (Untyped)。
    *   管理全局帧分配器。
    *   处理用户进程的缺页异常（延迟分配，COW）。
*   **进程与线程管理**:
    *   **ELF 加载**: 解析 ELF 二进制文件并设置初始 VSpace（虚拟地址空间）。
    *   **调度策略**: 虽然内核调度线程，但 Factotum 管理线程优先级和时间片分配（通过内核 capability 操作）。
    *   **生命周期**: 处理进程创建 (`spawn`) 和销毁 (`kill`)。
    *   **多线程**:
        *   管理线程组（共享相同 VSpace 的所有线程）。
        *   实现线程本地存储 (TLS) 设置。
        *   处理线程同步原语（类 futex 机制）。
*   **异常处理**:
    *   注册为其生成的所有进程的异常处理程序。
    *   处理段错误或非法指令等故障，可能将其转换为信号（对于 Tux）或错误报告。
*   **Capability 管理**:
    *   充当 capability 代理，铸造 capabilities 并将其分发给新进程。

## 3. 架构

Factotum 作为高优先级服务运行。它在自己的地址空间中维护所有运行进程的表（进程控制块 - PCB）。

### 与 9Ball 的交互
9Ball 引导 Factotum。一旦运行，Factotum 接管创建 9Ball 或其他服务请求的未来进程的责任。

### 与客户端的交互
客户端（如 Tux 或 Gopher）请求 Factotum：
*   `yield`: 放弃 CPU。
*   `map_memory`: 请求共享内存。
*   `spawn_thread`: 在现有 PD 中创建新线程。

## 4. 接口

Factotum 通过 **Factotum 协议** (ID `0x0100`) 暴露其功能。

### 4.1 进程生命周期
*   `SPAWN(name_ptr, name_len, flags) -> [pid]`
    *   从可执行文件创建新进程。
    *   **Flags**: 控制行为（例如 `WAIT`, `DEBUG`, `RFNOMNT`）。
        *   `RFNOMNT`: 如果设置，请求 Gopher 为子进程创建新的私有命名空间（Plan 9 风格）。
*   `EXIT(status) -> !`
    *   终止调用进程并通知父进程。
*   `WAIT(pid) -> [status]`
    *   阻塞直到指定的子进程终止。
    *   如果 `pid` 为 -1，等待任何子进程。
*   `KILL(pid, signal) -> [result]`
    *   向目标进程发送终止信号。
*   `FORK() -> [pid]`
    *   创建调用进程的副本（COW 语义）。

### 4.2 内存管理
*   `SBRK(increment) -> [new_brk]`
    *   调整进程的堆大小（程序断点）。
*   `MMAP(addr, len, prot, flags, fd, offset) -> [vaddr]`
    *   将文件或匿名内存映射到地址空间。
    *   支持共享内存创建。
*   `MUNMAP(addr, len) -> [result]`
    *   取消映射内存区域。

### 4.3 线程控制
*   `THREAD_CREATE(entry, stack, arg, tls_base) -> [tid]`
    *   在当前进程中创建新线程。
    *   设置初始执行上下文和 TLS。
*   `THREAD_EXIT(status) -> !`
    *   终止调用线程。
*   `THREAD_JOIN(tid) -> [status]`
    *   等待同一进程中的特定线程退出。
*   `FUTEX_WAIT(addr, val, timeout) -> [result]`
    *   原子地检查 `*addr == val`，如果是，则休眠。
*   `FUTEX_WAKE(addr, count) -> [woken_count]`
    *   唤醒在 `addr` 上等待的 `count` 个线程。
*   `YIELD() -> []`
    *   自愿放弃剩余的时间片。
*   `SLEEP(milliseconds) -> []`
    *   暂停执行指定的持续时间。

### 4.4 调试与检查
*   `GET_PID() -> [pid]`
    *   返回调用者的进程 ID。
*   `GET_PPID() -> [ppid]`
    *   返回父进程的进程 ID。
*   `PS() -> [buffer_cap]`
    *   返回包含进程列表快照的只读共享内存缓冲区。

### 4.5 Capability 代理
*   `REQUEST_CAP(service_id) -> []` (通过 IPC 传输 Cap)
    *   请求系统服务的句柄（例如 Gopher, Rio）。
    *   Factotum 在授予之前验证权限。
