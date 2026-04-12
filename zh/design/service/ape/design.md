# APE (ANSI/POSIX Environment) 设计文档

## 1. 简介

APE 是 Glenda 的 Linux ABI 兼容层，负责把 Linux 风格 syscall 语义转换为 Glenda 的 capability + IPC 语义。

在当前实现中，APE 以独立服务运行在 `service/ape/`，由内核故障转发（`KERNEL_PROTO/SYSCALL`）驱动 syscall 分发。

## 2. 当前实现（与代码一致）

### 2.1 Syscall 进入路径

当前主路径为：

1. 用户线程触发 Linux 风格 syscall；
2. 内核将上下文封装为 `KERNEL_PROTO` 消息；
3. APE 在 `handler.rs` 中按 syscall 号分发到 `syscall/{io,process,misc}.rs`；
4. APE 调用后端服务（Warren / FS / Prism 等）并回填返回值。

> 代码锚点：`service/ape/src/ape/fault.rs`、`service/ape/src/ape/handler.rs`

### 2.2 生命周期职责分层

- **Warren（资源所有者）**
  - 创建/销毁进程内核对象：`TCB/CNode/VSpace/UTCB/TrapFrame/KStack`；
  - 维护全局进程表与资源回收（`create`, `exit_wrapper`）。
- **APE（Linux 语义管理者）**
  - 维护 Linux 侧 PID/FD/路径与内存映射元数据；
  - 保存 `host_pid -> ape_pid` 映射；
  - 在 `exit/fault` 时先清理 APE 本地引用，再调用 `proc_client.kill`。

> 代码锚点：`service/warren/src/warren/process.rs`、`service/warren/src/warren/mod.rs`、`service/ape/src/ape/mod.rs`、`service/ape/src/ape/syscall/process.rs`

### 2.3 当前已实现 syscall 范围

当前 APE 映射包含：

- 进程/内存：`getpid/gettid/getppid/set_tid_address/exit/exit_group/brk/mmap/mprotect/munmap/execve/clone`；
- 文件与终端：`openat/close/read/write/readv/writev/lseek/ioctl`；
- 基础兼容：`uname/rt_sigaction/rt_sigprocmask/set_robust_list/prlimit64/clock_gettime/gettimeofday/nanosleep/getrandom/getuid/geteuid/getgid/getegid`。

> 代码锚点：`service/ape/src/ape/handler.rs`

## 3. 后端路由（当前）

| Linux 语义 | 后端服务 | 当前实现方式 |
| :--- | :--- | :--- |
| 进程对象生命周期 | Warren | `ProcessClient::create/get_cnode/kill` |
| 堆/页表相关能力分配 | Warren | `ResourceClient::alloc/free` + APE 侧映射策略 |
| 文件系统 I/O | FS（通过 `FsClient`） | APE 维护 fd 表；每个打开文件可使用独立 badged endpoint |
| 终端 I/O | Prism | `TERM_GET_STR/TERM_PUT_STR/TERM_IOCTL` |

## 4. 安全与性能白名单设计

本节定义“哪些操作必须经过 APE 代理（控制面）”与“哪些操作可下放为直连（数据面）”。

### 4.1 强制代理白名单（控制面，必须经 APE）

下列请求必须经过 APE，不允许应用直接持有等价高权限能力：

1. **生命周期控制**：`clone/execve/exit/exit_group`；
2. **地址空间策略**：`mmap/munmap/mprotect/brk`（需要维护 APE 的 `memory_maps/lazy_memory_maps` 一致性）；
3. **命名空间与路径策略**：`openat` 的路径解析（`cwd/root_dir`）与 tty 特殊路径判定；
4. **权限敏感 ioctl / 设备控制**；
5. **任何 capability 铸造、转移、注册类动作**（保留在 Warren + APE 控制面）。

**安全目标**：避免应用绕过 APE 状态机导致 pid/fd/映射失真，或直接升级资源权限。

### 4.2 可直连白名单（数据面，按能力下放）

在不破坏控制面一致性的前提下，允许对“已授权对象”的高频数据操作走直连：

1. **文件句柄数据 I/O**：对已打开文件句柄的 `read/write/readv/writev`；
2. **终端字节流 I/O**：`TERM_GET_STR/TERM_PUT_STR`（仅限被授予的 VT endpoint）；
3. **纯查询类本地调用**：如 `clock_gettime/gettimeofday` 可继续优化为更短路径（不改变权限模型）。

直连前提：

- 能力必须是 **最小权限** 且 **对象级别**（不授予全局服务管理能力）；
- 由 APE/Warren 分配并可撤销；
- 直连失败可回退到 APE 代理路径。

### 4.3 默认拒绝策略

未列入“可直连白名单”的请求，默认走 APE 代理。

即：

$$
	ext{DirectAllowed(op)} = op \in \text{Whitelist}_{data}
$$

否则：

$$
	ext{Route}(op) = \text{APE Proxy}
$$

## 5. 采用该白名单的性能/安全权衡

- **性能**：高频数据面减少一次或多次中转 IPC；APE 压力与尾延迟下降；
- **安全**：控制面仍集中，避免生命周期与权限模型分裂；
- **可维护性**：通过“默认代理 + 小范围直连白名单”控制复杂度，便于逐步演进。

## 6. 相关代码锚点

- APE syscall 分发：`service/ape/src/ape/handler.rs`
- APE I/O 路径：`service/ape/src/ape/syscall/io.rs`
- APE 进程/内存路径：`service/ape/src/ape/syscall/process.rs`
- APE fault 与生命周期清理：`service/ape/src/ape/fault.rs`
- Warren 进程创建与回收：`service/warren/src/warren/process.rs`、`service/warren/src/warren/mod.rs`

