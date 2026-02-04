# Tux Design Document

## 1. Introduction
Tux is the **Linux Compatibility Environment** for Glenda. Its architecture adopts the classic **L4Linux** (Single Server) pattern. Tux is essentially a ported Linux kernel running as a standard **User-mode Service** on Glenda, rather than inside a fully virtualized VM.

This architecture allows Linux applications to run alongside Glenda native services with near-native performance.

## 2. Core Architecture (L4Linux Model)

### 2.1 Single Server
The Tux service itself is a Linux kernel instance running on top of the microkernel.
*   **Linux Threads**: All Linux kernel threads run effectively as one or more threads on Glenda.
*   **User Space**: Linux applications (e.g., `bash`, `gcc`) run as independent Glenda tasks (with their own address spaces).

### 2.2 Paravirtualization Mechanism
Tux achieves paravirtualization by modifying the Linux kernel source code (porting it to the Glenda architecture), replacing underlying sensitive instructions and hardware access operations with **Hypercalls (HCall)** or Glenda IPC calls.
1.  **Instruction Replacement**: Privileged instructions (e.g., `sret`, `sfence.vma`, CSR access) are replaced with active calls to the Glenda microkernel or Tux Server.
2.  **HCall Interface**: Uses `ecall` (as HCall) to directly request Glenda services, avoiding the overhead of passive exception trapping.
3.  **Interrupt Emulation**: Hardware interrupts are delivered to Tux as IPC messages, which Tux handles within its event loop as logical interrupts.

## 3. Resource Mapping

Tux does not control hardware directly. Instead, it interacts with Glenda native services via "Stub Drivers".

*   **Block Device**: The Tux kernel includes a virtual block driver that translates block I/0 requests into IPC calls to **Fossil** or **Unicorn**.
*   **Network**: The Tux kernel includes a virtual NIC driver that sends network packets to **Gopher**.
*   **Console**: Mapped to **9Ball** or **Factotum** console output.
*   **Memory**: Tux acts as the **Pager (External Faulter)** for its client processes. When a Linux app triggers a page fault, Tux receives an IPC and maps memory by manipulating Glenda specific page table capabilities.

## 4. Management & Interoperability

Tux provides rich interfaces for interaction with the Glenda native environment:

*   **Control Plane (9P)**: Tux exports a 9P filesystem exposing internal Linux state (counterparts to `/proc`, `/sys`), allowing Glenda tools (`rc`, `ls`) to inspect Linux processes.
*   **Data Plane (Shared Memory)**: Efficient exchange of large data blocks with **Fossil** (FS) and **Gopher** (Net) via shared memory.
*   **Signals & Events**: A mechanism is defined allowing Glenda tasks to send signals to Linux processes and vice versa.

## 5. Relationship with Chimera

While Tux adopts the L4Linux architecture, Chimera (Virtualization Manager) still plays an auxiliary role:
*   If Tux needs to run in a hardware virtualization container (to support unmodified privileged instructions), Chimera acts as the underlying VMM.
*   In pure Para-virtualized mode, Tux runs directly on the microkernel without Chimera's intervention. The default configuration favors pure para-virtualization for lower overhead.

