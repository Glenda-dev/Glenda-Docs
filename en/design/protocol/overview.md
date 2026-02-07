# Interaction Protocols Design

## 1. Introduction
This section defines the communication protocols used within Glenda, corresponding to the definitions in `libglenda-rs/ipc/proto`.

## 2. Protocol Layers

Glenda uses a layered protocol approach:

### 2.1 Layer 0: Microkernel IPC (Raw)
*   **Mechanism**: seL4/Microkernel `Call`/`Reply`.
*   **Transport**: CPU Registers + UTCB (Message Registers).
*   **MsgTag**: First register, contains Protocol ID (Label) and Argument Count.

### 2.2 Layer 1: Transport Protocols

#### Ring Buffer (Data Plane)
Used for high-performance data transfer (e.g., Network Packets, Bulk Disk I/O).
*   **Structure**: Shared Memory Region containing a Descriptor Ring and Data Buffers.
*   **Signaling**: "Doorbell" IPCs (Notifications) are sent only when necessary to wake the peer.

### 2.3 Layer 2: Service Protocols

Specific command sets for core services, defined by unique **Protocol Labels**. See individual files for details.

| Protocol | Label Range | Description | Reference |
| :--- | :--- | :--- | :--- |
| **Generic** | `0x000` | Generic replies | [generic.md](generic.md) |
| **Kernel** | `0x100 - 0x1FF` | Kernel Core operations | [kernel.md](kernel.md) |
| **Process** (Warren) | `0x200 - 0x2FF` | Process Lifecycle | [process.md](process.md) |
| **Device** (Unicorn) | `0x300 - 0x3FF` | Device Management | [device.md](device.md) |
| **Init** | `0x400 - 0x4FF` | Startup protocols | [init.md](init.md) |
| **Fossil** | `0x500 - 0x5FF` | Filesystem Control | [fs.md](fs.md) |
| **Gopher** | `0x600 - 0x6FF` | Network Stack Control | [net.md](net.md) |
| **Chimera** | `0x700 - 0x7FF` | Virtualization Management | [virt.md](virt.md) |

## 3. Data Definitions (`libglenda-rs/ipc/proto`)

Refer to the specific protocol documents for struct definitions and command IDs.

### 3.1 Common Types
Definitions for `Stat` structs used across the system.
