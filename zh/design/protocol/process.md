# 进程协议定义 (Process Protocol)

## Protocol Label Range
`0x200 - 0x2FF`

## 描述
用于控制进程生命周期、内存状态和线程的协议。由 **Warren** 管理。

## 指令 (Commands)

### 生命周期 (Lifecycle)
*   `CREATE` (`0x01`): 创建一个空进程对象并返回 PID。
*   `SPAWN` (`0x02`): 创建并加载可执行镜像。
*   `EXIT` (`0x03`): 终止当前进程。
*   `KILL` (`0x04`): 终止目标进程。

### 线程控制 (Thread Control)
*   `THREAD_CREATE` (`0x10`): 在目标进程内创建线程。
*   `THREAD_EXIT` (`0x11`): 线程退出（预留）。
*   `THREAD_JOIN` (`0x12`): 等待线程结束（预留）。
*   `YIELD` (`0x15`): 让出 CPU（预留）。

### 调试与检查 (Debugging & Inspection)
*   `GET_CNODE` (`0x20`): 获取目标进程 CNode 能力（按权限检查）。

## 说明

- `WAIT/FORK/MMAP/MUNMAP/SBRK` 等 Linux 风格语义不属于 `PROCESS_PROTO` 指令集本体；
- 这些语义通常由 APE 在 Linux ABI 层实现，并通过 `PROCESS_PROTO + RESOURCE_PROTO` 组合完成底层资源操作。
