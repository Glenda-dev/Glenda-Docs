# LINE Design Document

## 1. Introduction
LINE (LINE Is Not Emulator) is a Linux Compatibility Layer for Glenda. It allows unmodified Linux ELF binaries to run on Glenda by trapping Linux system calls and translating them into Glenda native operations.

## 2. Responsibilities

*   **Binary Loading**:
    *   Recognizes Linux ELF binaries.
    *   Sets up the Linux-compatible stack and auxiliary vector (AT_SYSINFO).
*   **Syscall Translation**:
    *   Intercepts the `ecall` (RISC-V) instruction used by Linux binaries.
    *   Translates Linux syscall numbers (e.g., `sys_write`, `sys_open`) into IPC calls to Tux (for POSIX compliance) or Gopher/Factotum.
*   **Path Translation**:
    *   Maps Linux paths (e.g., `/proc`, `/sys`) to Glenda equivalents.

## 3. Architecture

LINE is likely implemented as a library or a special loader that runs within the process address space of the Linux program, or as a service that handles the trap.

### Execution Flow
1.  User runs `./linux_app`.
2.  Factotum detects the Linux ABI.
3.  Factotum injects the LINE trampoline/shim into the process.
4.  When `linux_app` executes `ecall`:
    *   The trap is caught by the kernel.
    *   The kernel forwards the exception to Factotum.
    *   Factotum redirects execution to the LINE shim in the user process.
    *   LINE shim parses registers (a0-a7), determines the syscall.
    *   LINE calls Tux/Gopher.
    *   LINE returns the result to `linux_app` and resumes execution.

## 4. Goals
*   Run basic Linux CLI tools (busybox, gcc).
*   Eventually support more complex runtimes (Python, Node.js).
