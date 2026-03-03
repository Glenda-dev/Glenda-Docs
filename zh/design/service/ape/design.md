# APE (ANSI/POSIX Environment) 设计文档

## 1. 简介
APE 是 Glenda 的 **Linux 二进制兼容性 (Linux ABI Emulator)** 与 **POSIX 兼容垫片层 (Shim)**。其核心目标是实现在 Glenda 微内核上直接运行未经修改的 Linux 二进制程序。

APE 采用多层拦截架构，结合了 **vDSO (极速路径)**、**动态库拦截 (快速路径)** 以及 **内核 Fault Handler (保底路径)**。

## 2. 总体架构设计

### 2.1 三层系统调用拦截机制 (Three-Tier Interception)

为了兼顾兼容性与性能，APE 提供三种不同层级的系统调用转换路径：

- **极速路径 (vDSO)**：
  - **机制**：在进程地址空间映射一个兼容 Linux vDSO ABI 的动态库。
  - **实现**：无需陷入内核，直接在用户态读取与系统服务（如 March 时间服务）共享的**只读内存页**。适用于 `gettimeofday`, `clock_gettime` 等高频只读调用。
  - **优势**：零上下文切换开销。

- **快速路径 (动态库拦截)**：
  - **机制**：对于动态链接的 Linux 程序，通过注入 `libape.so` 拦截 `glibc` 或 `musl` 封装的系统调用函数。
  - **实现**：`libape` 直接将 POSIX 请求转换为 Glenda 原生的 IPC 请求，发送至 APE 服务或目标功能服务（如 Fossil）。
  - **优势**：高性能，避免了触发硬件 Syscall 异常。

- **慢速/保底路径 (Fault Handler)**：
  - **机制**：针对静态编译或使用内联汇编触发真实 `syscall/ecall` 指令的行为。
  - **实现**：内核捕获硬件异常后，通过 **Fault Handler** 机制将执行流委托给 APE 服务。APE 服务解析寄存器状态，模拟执行该系统调用，并修改被捕获线程的上下文（PC+4, 返回值寄存器）后恢复执行。
  - **优势**：最大化兼容性。

### 2.2 核心组件

- **APE Service (独立服务端)**：
  - 作为一个独立的 Glenda 系统服务运行（`service/ape/`）。
  - **状态维护**：维护 Linux 进程树、PID 映射、全局 POSIX 信号挂起队列。
  - **资源映射**：管理 Linux `int fd` 到 Glenda `CapPtr` 的转换。

- **libape (用户态库)**：
  - 集成在进程空间内，负责 IPC 通信的封装。
  - **VFS 垫片**：在用户空间维护轻量级状态（CWD、每进程挂载点）。

## 3. 服务路由映射

APE 将 Linux 的资源请求路由到相应的 Glenda 原生服务：

| Linux 资源 | 后端服务 | 协议 | 实现方式 |
| :--- | :--- | :--- | :--- |
| **文件系统 (FS)** | **Fossil** | FS 协议 + SHM | FD 映射到文件 Capability |
| **网络 (Network)** | **Gopher** | Socket IPC | APE 代理 Socket 状态 |
| **硬件/设备** | **Unicorn** | 设备特定 IPC | 映射为后端 Virtio 设备 |
| **进程/内存**| **Warren** | Process/Mem | 封装 `fork`, `exec`, `sbrk` |
| **时间/定时器**| **March** | vDSO / Timer | 共享内存 / 信号通知 |

## 4. 关键原语实现

### 4.1 信号模拟 (Signal Emulation)
- **注册**：程序通过 APE 向 Warren 注册一个专用的回复端点（Reply Endpoint）。
- **投递**：当 APE 服务确定需要投递信号时，通过该端点发送 IPC，APE 库在用户态接管执行流，模拟执行 POSIX 信号处理函数。

### 4.2 进程克隆 (Fork/Exec)
- `fork()`: 调用 Warren 创建新进程，利用内核的地址空间克隆及 Capability 复制。
- `exec()`: 由 APE 请求 Warren 重新解析 ELF 头部。如果是 Linux ELF，Warren 会自动将 APE 服务关联为该进程的异常处理程序。

## 5. 性能优化
*   **零拷贝 I/O**: 对于大块数据传输，APE 自动在进程空间与服务间建立共享内存映射。
*   **vDSO 共享**: 全系统共享同一个只读时间/状态页，减少内核内存占用。

