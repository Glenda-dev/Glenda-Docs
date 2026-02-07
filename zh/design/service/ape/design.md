# APE (ANSI/POSIX Environment) 设计文档

## 1. 简介
APE 是 Glenda 的 **POSIX 兼容垫片层 (Shim)**。与传统的单体 POSIX 服务器不同，APE 主要实现为 **客户端库**（属于 `libglenda` 或 `musl-glenda` 的一部分）。它拦截标准的 POSIX 系统调用，并将其指向相应的 Glenda 原生服务。

## 2. 架构

APE 作为运行在应用程序进程空间内的“智能代理”发挥作用。

### 2.1 库层 (Library Layer)
*   **位置**: `lib/libglenda-rs/src/ape/` 或集成在 `musl-glenda` 的系统调用后端中。
*   **功能**: 将 C-ABI 系统调用/libc 函数转换为 Glenda IPC 消息。

### 2.2 服务路由
APE 在进程的用户空间内存中维护一个路由表，将文件描述符 (FD) 映射到 Glenda Capabilities (Endpoints)。

| FD 范围 | 类型 | 后端服务 | 协议 |
| :--- | :--- | :--- | :--- |
| `0, 1, 2` | 标准IO | **Warren** / **Rio** | Ring Buffer / 序列化 IO |
| `3...N` | 文件 | **Fossil** | 9P2000 + SHM |
| `M...P` | 套接字 | **Gopher** | APE 专用 Socket IPC |
| `X...Y` | 设备 | **Unicorn** | 设备特定 IPC |

## 3. 关键组件

### 3.1 VFS 垫片 (Virtual File System)
Glenda 应用程序不与中央 VFS 内核对象对话。APE 在用户空间维护轻量级的 VFS 状态。
*   **CWD**: 当前工作目录跟踪。
*   **挂载表**:每进程挂载点（如果不完全使用 9P 命名空间服务器）。
*   **FD 表**: 映射 `int fd` 到 `{ Capability, Offset, Flags }` 的 `Vec<FileEntry>`。

### 3.2 信号模拟 (Signal Emulation)
*   **接收器**: 在 **Warren** 注册一个 IPC 端点以接收“软件中断”。
*   **分发器**: 当信号 IPC 到达时，APE 中断主线程（或使用辅助线程）以执行注册的 POSIX 信号处理程序。

### 3.3 进程原语
*   `fork()`: 通过 **Warren** 实现（创建新任务，COW 内存）。
*   `exec()`: 通过 **Warren** 加载新的 ELF。

## 4. 使用模式

1.  **原生链接**: Glenda 感知的 Rust 应用直接使用 `libglenda`（绕过部分 APE，使用原生 Trait）。
2.  **遗留链接**: C/C++ 应用链接到 `musl-glenda` (APE 后端)。
3.  **二进制翻译**: 纯 Linux 二进制文件在 **Tux** 下运行，Tux 充当“服务端 APE”。

## 5. 性能
*   **零拷贝 I/O**: APE 对大于 512 字节的 `read/write` 负载使用共享内存，以避免 IPC 拷贝开销。
*   **批处理系统调用**: (计划中) 聚合多个小型元数据操作。
