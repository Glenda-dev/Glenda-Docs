# Tux 设计文档

## 1. 简介
Tux 是 Glenda 的 POSIX 服务器。由于 Glenda 是一个具有自己原生异步 IPC 的微内核，它本身不支持 POSIX API（fork, exec, 阻塞 I/O, 信号）。Tux 通过为用户应用程序提供符合 POSIX 标准的环境来弥补这一差距。

## 2. 职责

*   **POSIX API 实现**:
    *   提供标准 C 库接口（通过与 Tux 对话的 `musl-glenda`）。
    *   处理 `fork()`, `exec()`, `wait()`, `pipe()`, `kill()`。
*   **信号处理**:
    *   模拟 Unix 信号。Factotum 通知 Tux 硬件故障，Tux 将其作为信号 (SIGSEGV, SIGILL) 传递给注册的进程。
*   **进程组管理**:
    *   管理 PID, PGID, 会话。
*   **文件描述符表**:
    *   维护整数 FD 和 Glenda Capability 句柄（用于 Gopher 文件、套接字、管道）之间的映射。

## 3. 架构

Tux 作为高优先级用户空间服务器运行。与 `musl-glenda` (libc) 链接的应用程序将系统调用指令转换为对 Tux 的 IPC 调用。

### 通过 Badge 进行身份验证
为了确保安全，Tux 依赖于 **IPC Badges**。当 Tux 将会话 Capability 分发给新进程时，它会将一个唯一的、不可变的 Badge 附加到 Endpoint。
- 当 Tux 收到消息时，内核将 Badge 注入接收者的上下文。
- Tux 使用此 Badge 在其内部进程表中查找 `ProcessContext`，防止进程欺骗其 PID。

### "Fork" 问题
`fork()` 在微内核中很难实现。Tux 与 Factotum 合作实现写时复制 (COW) 分叉：
1.  Tux 请求 Factotum 克隆地址空间（将页面标记为只读）。
2.  Tux 复制文件描述符表。
3.  Tux 通过 Factotum 创建新线程。

## 4. 接口

*   **接口**: `org.glenda.posix.Tux`
*   **方法**:
    *   `Syscall(num: u32, args: [u64; 6]) -> u64`
    *   `RegisterSignalHandler(signum: u32, handler: u64)`
    *   `GetPid() -> u32`

## 5. IPC 协议
在 `libglenda-rs` 中定义，供 Tux 和应用程序共享使用。

```rust
#[repr(u32)]
pub enum PosixSyscall {
    Open = 1,
    Read = 2,
    Write = 3,
    Close = 4,
    Fork = 5,
    Exec = 6,
    Wait = 7,
    Exit = 8,
    GetPid = 9,
}
```

### 3.2 内部进程跟踪
Tux 维护所有活动 POSIX 进程的私有表。

```rust
struct PosixProcess {
    pid: i32,
    parent_pid: i32,
    // 微内核 Capabilities
    tcb_cap: CapPtr,
    vspace_cap: CapPtr,
    cspace_cap: CapPtr,
    // FD 表: FD -> 远程服务 Endpoint
    fd_table: BTreeMap<i32, CapPtr>,
    // 信号状态
    signal_notif: CapPtr,
    pending_signals: u64,
}
```

## 4. 安全与隔离

- **基于 Capability**: Tux 仅授予进程被授权访问的 FD (Capabilities)。
- **无全局状态**: 所有 POSIX 状态都封装在 Tux 内部。内核不知道“进程”或“文件”。
- **资源核算**: Tux 使用自己的 `Untyped` 内存池为新进程分配元数据，确保一个进程无法通过创建过多的子进程来耗尽系统范围的内存。
