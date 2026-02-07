# Process Protocol Definition

## Protocol Label Range
`0x200 - 0x2FF`

## Description
Protocol for controlling process lifecycle, memory state, and threading. Managed by **Warren**.

## Commands

### Lifecycle
*   `SPAWN` (1): Create a new process.
*   `EXIT` (2): Terminate current process.
*   `WAIT` (3): Wait for a child process to exit.
*   `KILL` (4): Terminate another process.
*   `FORK` (5): Fork current process (COW).

### Memory Management
*   `SBRK` (6): Adjust program break.
*   `MMAP` (7): Map memory pages.
*   `MUNMAP` (8): Unmap memory pages.

### Thread Control
*   `THREAD_CREATE` (9): Create a new thread in the current process.
*   `THREAD_EXIT` (10): Exit current thread.
*   `THREAD_JOIN` (11): Join a thread.
*   `FUTEX_WAIT` (12): Wait on a futex.
*   `FUTEX_WAKE` (13): Wake futex waiters.
*   `YIELD` (14): Yield CPU.
*   `SLEEP` (15): Sleep for a duration.

### Debugging & Inspection
*   `GET_PID` (16): Get current Process ID.
*   `PS` (17): List processes.
