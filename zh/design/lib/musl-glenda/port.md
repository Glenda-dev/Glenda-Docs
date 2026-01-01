# Musl Libc 移植指南

本指南描述了将 Musl Libc 移植到 Glenda 微内核的过程。

## 1. 概述

Musl 是一个轻量级、快速且符合标准的 C 标准库实现。移植 Musl 使得 Glenda 能够支持标准的 C/C++ 应用程序。

## 2. 移植策略

我们将创建一个名为 `musl-glenda` 的 Musl 分支，并添加对 Glenda 的支持。

### 2.1 目录结构

在 Musl 源码树中：
- `arch/riscv64/`: 架构相关的汇编代码（通常不需要修改，除非涉及特定的系统调用约定）。
- `src/internal/glenda/`: Glenda 特定的内部实现。

### 2.2 系统调用适配

Musl 依赖于 Linux 风格的系统调用。我们需要拦截这些系统调用并将其转换为 Glenda 的 IPC 调用。

主要涉及的文件：
- `arch/riscv64/syscall_arch.h`: 定义 `__syscall_N` 宏。

我们将修改 `__syscall_N` 宏，使其调用 `libglenda` 提供的系统调用处理函数，而不是执行 `ecall` 指令（或者在 `ecall` 处理程序中进行转换，但在用户态转换更灵活）。

**方案 A: 用户态模拟 (推荐)**
在 `syscall_arch.h` 中，将 `__syscall` 重定向到一个名为 `glenda_syscall_handler` 的函数。该函数根据系统调用号（如 `SYS_write`, `SYS_open`）构造相应的 Glenda IPC 消息，发送给相应的服务（如 Factotum, Gopher）。

```c
// arch/riscv64/syscall_arch.h

long glenda_syscall_handler(long n, long a1, long a2, long a3, long a4, long a5, long a6);

#define __syscall(n, a1, a2, a3, a4, a5, a6) \
    glenda_syscall_handler(n, a1, a2, a3, a4, a5, a6)
```

### 2.3 关键系统调用实现

以下是必须实现的最小系统调用集：

- **进程控制**: `exit`, `exit_group`.
- **内存管理**: `brk`, `mmap`, `munmap`.
- **文件 I/O**: `read`, `write`, `open`, `close`, `fstat`.
- **线程**: `clone`, `futex` (用于同步).

#### `write` (stdout)
最初，`write(1, ...)` 可以通过内核的调试打印系统调用实现，或者发送给 Log 服务。

#### `mmap` / `brk`
需要与 Glenda 的内存管理服务交互，分配和映射内存页。

#### `open` / `read`
需要与 VFS 服务 (Factotum) 交互。

## 3. 构建系统

我们需要配置 Musl 的构建系统以使用我们的交叉编译器和头文件。

```bash
./configure \
    --target=riscv64-unknown-none-elf \
    --prefix=/usr/local/musl \
    --disable-shared \
    --enable-static
```

可能需要修改 `config.mak` 或 `Makefile` 以链接 `libglenda`。

## 4. 启动代码 (crt1.o)

Musl 的 `crt1.o` 负责程序的初始化。我们需要修改它以适应 Glenda 的进程启动协议。
- 获取 UTCB 指针。
- 初始化 TLS。
- 解析命令行参数和环境变量（如果通过 IPC 传递）。

## 5. 线程支持 (Pthreads)

Musl 的 pthread 实现依赖于 `clone` 和 `futex`。
- **`clone`**: 映射到 Glenda 的线程创建操作。
- **`futex`**: 需要内核支持或在用户态通过 IPC 实现（效率较低）。Glenda 内核可能需要提供类似 futex 的原语。

## 6. 待办事项

- [ ] 建立 `musl-glenda` 仓库。
- [ ] 修改 `syscall_arch.h` 挂钩系统调用。
- [ ] 实现 `glenda_syscall_handler` 分发器。
- [ ] 实现基本的 `write` (stdout)。
- [ ] 实现 `exit`。
- [ ] 链接 `libglenda`。
- [ ] 运行 "Hello World"。
