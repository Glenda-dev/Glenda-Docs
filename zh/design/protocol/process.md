# 进程协议定义 (Process Protocol)

## Protocol Label Range
`0x200 - 0x2FF`

## 描述
用于控制进程生命周期、内存状态和线程的协议。由 **Warren** 管理。

## 指令 (Commands)

### 生命周期 (Lifecycle)
*   `SPAWN` (1): 创建新进程。
*   `EXIT` (2): 终止当前进程。
*   `WAIT` (3): 等待子进程退出。
*   `KILL` (4): 终止其他进程。
*   `FORK` (5): Fork 当前进程 (COW)。

### 内存管理 (Memory Management)
*   `SBRK` (6): 调整程序断点 (Heap)。
*   `MMAP` (7): 映射内存页。
*   `MUNMAP` (8): 取消映射内存页。

### 线程控制 (Thread Control)
*   `THREAD_CREATE` (9): 在当前进程中创建新线程。
*   `THREAD_EXIT` (10): 退出当前线程。
*   `THREAD_JOIN` (11): 等待线程结束。
*   `FUTEX_WAIT` (12): 等待 Futex。
*   `FUTEX_WAKE` (13): 唤醒 Futex 等待者。
*   `YIELD` (14): 让出 CPU。
*   `SLEEP` (15): 睡眠一段时间。

### 调试与检查 (Debugging & Inspection)
*   `GET_PID` (16): 获取当前进程 ID。
*   `PS` (17): 列出进程。
