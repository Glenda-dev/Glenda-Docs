# LINE 设计文档

## 1. 简介
LINE (LINE Is Not Emulator) 是 Glenda 的 Linux 兼容层。它允许未修改的 Linux ELF 二进制文件在 Glenda 上运行，方法是捕获 Linux 系统调用并将其转换为 Glenda 原生操作。

## 2. 职责

*   **二进制加载**:
    *   识别 Linux ELF 二进制文件。
    *   设置 Linux 兼容的堆栈和辅助向量 (AT_SYSINFO)。
*   **系统调用转换**:
    *   拦截 Linux 二进制文件使用的 `ecall` (RISC-V) 指令。
    *   将 Linux 系统调用号（例如 `sys_write`, `sys_open`）转换为对 Tux（用于 POSIX 合规性）或 Gopher/Factotum 的 IPC 调用。
*   **路径转换**:
    *   将 Linux 路径（例如 `/proc`, `/sys`）映射到 Glenda 等效路径。

## 3. 架构

LINE 可能实现为一个库，或者一个在 Linux 程序进程地址空间内运行的特殊加载器，或者一个处理陷阱的服务。

### 执行流程
1.  用户运行 `./linux_app`。
2.  Factotum 检测到 Linux ABI。
3.  Factotum 将 LINE trampoline/shim 注入到进程中。
4.  当 `linux_app` 执行 `ecall` 时：
    *   陷阱被内核捕获。
    *   内核将异常转发给 Factotum。
    *   Factotum 将执行重定向到用户进程中的 LINE shim。
    *   LINE shim 解析寄存器 (a0-a7)，确定系统调用。
    *   LINE 调用 Tux/Gopher。
    *   LINE 将结果返回给 `linux_app` 并恢复执行。

## 4. 目标
*   运行基本的 Linux CLI 工具 (busybox, gcc)。
*   最终支持更复杂的运行时 (Python, Node.js)。
