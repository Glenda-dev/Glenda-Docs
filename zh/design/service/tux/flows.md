# 交互流程

## 文件打开 (`open`)
1. **App** 在 libc 中调用 `open("/etc/motd")`。
2. **libc** 通过 IPC 向 Tux 发送 `PosixRequest`。
3. **Tux** 收到请求并通过其 **Badge** 识别调用者。
4. **Tux** 将路径查找转发给 **Gopher (VFS)**。
5. **Gopher** 返回文件会话的新的 `Endpoint Capability`。
6. **Tux** 分配一个新的 FD（例如 `3`），将 Capability 存储在进程的 `fd_table` 中，并将 `3` 返回给 App。

## 进程创建 (`fork`)
1. **App** 调用 `fork()`。
3. **Tux** 收到请求并通过其 **Badge** 识别调用者。
4. **Tux** 将路径查找转发给 **Factotum (进程控制)**。
5. **Factotum** 克隆地址空间 (COW)，创建一个新的进程条目，并将新的 PID 返回给 Tux。
6. **Tux** 为新进程复制 FD 表。
7. **Tux** 通过 Factotum 在新进程中创建一个新线程。
8. 父进程和子进程都从 `fork()` 调用恢复执行，子进程收到 `0`，父进程收到子进程的 PID。
