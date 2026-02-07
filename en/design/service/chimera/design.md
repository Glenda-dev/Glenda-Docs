# Chimera Design Document

## 1. Introduction
Chimera is the **Virtualization Manager** (VMM) for Glenda. It abstracts hardware virtualization extensions (such as RISC-V H-Extension or Intel VT-x) to provide secure execution environments for Guest OS kernels, Paravirtualized systems, and Hardware-isolated Containers.

## 2. Responsibilities

*   **Helper for HVM**: Supports Hardware Virtual Machines.
*   **Resource Partitioning**: Allocates Stage-2 Page Tables (Guest Physical Address to Host Physical Address translation).
*   **vCPU Scheduling**: Maps Guest vCPUs to Glenda Threads managed by Warren.
*   **Device Emulation**: Optionally provides device models (typically delegated to user-space backend drivers like Tux or Unicorn).

## 3. Architecture

Chimera runs as a high-privilege service (or Driver Component).

### 3.1 Virt-Extension Management
Chimera is the only component allowed to invoke virtualization-specific commands (like `hfence`, `hlv`).
*   **VMID Management**: Allocates hardware VMIDs.
*   **Interrupt Injection**: Injects virtual interrupts into the Guest.

### 3.2 Operating Modes

#### A. Full Virtualization (HVM)
Runs an unmodified Guest OS (e.g., Windows, Standard Linux).
*   **Hardware Emulation**: Relies on **Unicorn** to emulate a complete set of hardware devices (UART, Block, Net).
*   **Isolation**: Strict isolation using Stage-2 page tables.

#### B. Paravirtualization (PV)
Runs a modified Guest OS aware of the Glenda environment (e.g., specialized RTOS, or Tux in HVM mode).
*   **Hypercalls**: Guest uses `hVCALL` to request services directly from Chimera/Glenda.
*   **VirtIO**: Uses VirtIO over MMIO for efficient I/O.

#### C. Containerization (MicroVM)
Provides hardware-enforced isolation for standard Glenda/Linux processes (Kata Containers style).
*   **Lightweight**: Minimal memory footprint, shared read-only pages.
*   **Filesystem Passthrough**: Uses `virtio-fs` to map host directories directly.
*   **Fast Boot**: Optimized for sub-second startup to run a single payload.

## 4. Interfaces

### 4.1 Management Interface (IPC)
*   `create_vm(config) -> Cap`: Creates a new VM instance.
*   `map_memory(vm_cap, gpa, hpa, size, perms)`: Configures Stage-2 mapping.
*   `create_vcpu(vm_cap) -> ThreadCap`: Creates a vCPU thread.

### 4.2 Trap Forwarding protocol
When a Guest triggers a VMExit that Chimera cannot handle in-kernel (e.g., Device IO), Chimera sends an **Fault IPC** to the registered handler endpoint (usually **Unicorn** or a specific device model service).
