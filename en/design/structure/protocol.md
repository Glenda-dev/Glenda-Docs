# Glenda Protocol Interface Specification

This document defines the protocol interfaces for communication between various components in the Glenda system.

## 1. IPC Message Format

Glenda's IPC messages are passed via the UTCB (User Thread Control Block) and registers.

### 1.1 Message Tag (MsgTag)

Each message starts with a `MsgTag`, typically located in register `MR0` (or architecture-specific argument registers).

```rust
struct MsgTag {
    label: u16,   // Protocol Label (Protocol ID)
    flags: u4,    // Flags (e.g., HAS_CAP)
    length: u4,   // Message Length (Number of registers)
}
```

### 1.2 Protocol Labels

Labels are used to distinguish different service requests or event types.

#### Kernel Reserved Labels (Kernel Protocols)
| Label | Name | Description |
| :--- | :--- | :--- |
| `0xFFFF` | `PAGE_FAULT` | Page Fault Notification |
| `0xFFFE` | `EXCEPTION` | General Exception Notification |
| `0xFFFD` | `UNKNOWN_SYSCALL` | Unknown Syscall |
| `0xFFFC` | `CAP_FAULT` | Capability Fault |
| `0xFFFB` | `IRQ` | Hardware Interrupt Notification |
| `0xFFFA` | `NOTIFY` | Asynchronous Notification (Badge Only) |

#### System Service Labels (System Service Protocols)
| Range | Service | Description |
| :--- | :--- | :--- |
| `0x0000` - `0x00FF` | **Generic** | Generic Control Protocols (Ping, Debug) |
| `0x0100` - `0x01FF` | **Factotum** | Process and Thread Management |
| `0x0200` - `0x02FF` | **Gopher** | File System and IO |
| `0x0300` - `0x03FF` | **Unicorn** | Device Driver Control |
| `0x0400` - `0x04FF` | **Rio** | Graphics Display Protocol |

## 2. Detailed Protocol Definitions

### 2.1 Factotum Protocol (Process & Memory)

Factotum acts as the central process manager and exception handler.

**Base Protocol ID**: `0x0100`

| ID | Method | Arguments (UTCB/Regs) | Return Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x0101` | `SPAWN` | `[name_ptr, name_len]` | `[pid]` | Spawns a new process from a filesystem path. |
| `0x0102` | `EXIT` | `[status]` | - | Terminates the calling process. |
| `0x0103` | `WAIT` | `[pid]` | `[status]` | Blocks until the specified child process exits. |
| `0x0104` | `YIELD` | `[]` | `[]` | Relinquishes the current timeslice. |
| `0x0105` | `SBRK` | `[increment]` | `[new_break]` | Increases the process heap size. |
| `0x0106` | `MAP_DEVICE` | `[paddr, size, flags]` | `[vaddr]` | Maps a physical device region (requires privileges). |
| `0x0107` | `GET_PID` | `[]` | `[pid]` | Returns the current process ID. |

### 2.2 Gopher Protocol (9P2000)

Gopher implements the standard **9P2000** protocol. Glenda IPC serves as the transport layer, replacing the traditional TCP or pipe transport. This allows for network transparency and a unified resource interface.

**Transport Mechanism:**
*   **Request**: The client constructs a standard 9P `T-message` and writes it into the **UTCB (IPC Buffer)**.
*   **IPC Call**: The client invokes `Call` on Gopher's Endpoint.
    *   **Label**: `0x0200` (9P_REQUEST)
    *   **Args**: `[msg_length, 0, 0, 0, 0, 0]`
*   **Response**: Gopher processes the request and writes the corresponding 9P `R-message` back into the client's UTCB.

**Supported 9P Operations:**

| 9P Message Type | Description | Glenda Implementation Note |
| :--- | :--- | :--- |
| `Tversion` / `Rversion` | Protocol version negotiation | Negotiates buffer size (msize) |
| `Tauth` / `Rauth` | Authentication | Can use Capabilities for auth |
| `Tattach` / `Rattach` | Establish connection to root | Returns the root `fid` |
| `Twalk` / `Rwalk` | Traverse directory hierarchy | Resolves paths component by component |
| `Topen` / `Ropen` | Open file | Prepares `fid` for I/O |
| `Tcreate` / `Rcreate` | Create file | Creates file in parent directory |
| `Tread` / `Rread` | Read data | Data returned in UTCB (or Shared Mem) |
| `Twrite` / `Rwrite` | Write data | Data sent via UTCB (or Shared Mem) |
| `Tclunk` / `Rclunk` | Close fid | Releases the handle |
| `Tremove` / `Rremove` | Remove file | Deletes the file |
| `Tstat` / `Rstat` | Get attributes | Retrieves file metadata |
| `Twstat` / `Rwstat` | Set attributes | Updates file metadata |

**Large I/O Optimization (Zero-Copy):**
Since the UTCB size is limited (typically 4KB), standard `Tread`/`Twrite` are inefficient for large data.
*   **Shared Memory**: Clients can pass a Frame Capability (Shared Memory) to Gopher.
*   **Bulk IO**: A specialized extension or a separate `READ_BULK` / `WRITE_BULK` method can be used to transfer data directly to/from the shared frame, bypassing the message copy.

### 2.3 Unicorn Protocol (Device Drivers)

Unicorn manages device discovery, interrupt routing, and DMA memory allocation.

**Base Protocol ID**: `0x0300`

| ID | Method | Arguments | Return Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x0301` | `REGISTER` | `[device_id, type]` | `[driver_id]` | Registers a new driver instance. |
| `0x0302` | `IRQ_ACK` | `[irq_num]` | `[]` | Acknowledges an interrupt has been handled. |
| `0x0303` | `DMA_ALLOC` | `[size]` | `[paddr, vaddr]` | Allocates DMA-capable contiguous memory. |
| `0x0304` | `DMA_FREE` | `[paddr]` | `[]` | Frees DMA memory. |

### 2.4 Rio Protocol (Wayland Compositor)

Rio implements a **Wayland-compatible** compositor. It uses Glenda IPC as the transport layer for the Wayland wire protocol, replacing Unix domain sockets.

**Transport Mechanism:**
*   **Wire Protocol**: Standard Wayland object-oriented protocol.
*   **Shared Memory**: Used for command buffers (high throughput) and pixel buffers (`wl_shm`).
*   **Capabilities**: Used to pass shared memory handles (replacing file descriptors).

**Base Protocol ID**: `0x0400`

| ID | Method | Arguments | Return Values | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x0401` | `CONNECT` | `[]` | `[conn_id]` | Establishes a new Wayland client connection. |
| `0x0402` | `DISPATCH` | `[conn_id]` | `[]` | Notifies Rio to process messages in the shared command buffer. |
| `0x0403` | `GET_EVENT` | `[conn_id]` | `[has_event]` | Checks/Waits for events from the compositor. |
| `0x0404` | `PASS_CAP` | `[conn_id, obj_id]` | `[]` | Transfers a Capability (e.g., Frame) associated with a protocol object. |

**Key Wayland Globals Supported:**
*   `wl_compositor`: Surface creation.
*   `wl_shm`: Shared memory buffers.
*   `wl_seat`: Input devices (pointer, keyboard).
*   `xdg_wm_base`: Desktop window management.

### 2.5 Fault Handler Protocol

Messages sent by the kernel to the registered Fault Handler (usually Factotum) when a thread exception occurs.

**Page Fault (Label: 0xFFFF)**
*   **Sender**: Kernel
*   **Receiver**: Factotum
*   **Payload**:
    *   `Arg0`: `scause` (Exception Cause)
    *   `Arg1`: `stval` (Faulting Virtual Address)
    *   `Arg2`: `sepc` (Exception Program Counter)

**General Exception (Label: 0xFFFE)**
*   **Sender**: Kernel
*   **Receiver**: Factotum
*   **Payload**:
    *   `Arg0`: `scause`
    *   `Arg1`: `stval`
    *   `Arg2`: `sepc`

## 3. Process Capability Space (CSpace) Layout

To communicate with system components, a process must possess the corresponding **Endpoint Capabilities**. When Factotum spawns a new process, it populates the new process's CSpace with these "Well-known Capabilities".

| CPtr (Index) | Name | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | `NULL` | - | Reserved / Invalid |
| `1` | `TCB_SELF` | TCB | Capability to self TCB |
| `2` | `CNODE_SELF` | CNode | Capability to self CNode |
| `3` | `VSPACE_SELF` | PageTable | Capability to self VSpace (Root PageTable) |
| `4` | `EP_FACTOTUM` | Endpoint | **IPC Channel to Factotum** (Process Mgr) |
| `5` | `EP_GOPHER` | Endpoint | **IPC Channel to Gopher** (VFS Root) |
| `6` | `EP_UNICORN` | Endpoint | IPC Channel to Unicorn (Optional/Driver only) |
| `7` | `EP_RIO` | Endpoint | IPC Channel to Rio (Optional/GUI App only) |
| `10` | `FD_STDIN` | Endpoint/File | Standard Input |
| `11` | `FD_STDOUT` | Endpoint/File | Standard Output |
| `12` | `FD_STDERR` | Endpoint/File | Standard Error |

## 4. Interface Definition (Rust Trait Example)

```rust
/// Factotum Client Interface (Process Management)
pub trait ProcessManager {
    fn spawn(&self, name: &str) -> Result<Pid, Error>;
    fn exit(&self, status: usize) -> !;
    fn wait(&self, pid: Pid) -> Result<usize, Error>;
    fn yield_cpu(&self);
    fn sbrk(&self, increment: isize) -> Result<VirtAddr, Error>;
    fn map_device(&self, paddr: PhysAddr, size: usize, flags: usize) -> Result<VirtAddr, Error>;
    fn get_pid(&self) -> Pid;
}

/// Gopher Client Interface (9P2000 Abstraction)
pub trait FileSystem {
    fn attach(&self, uname: &str, aname: &str) -> Result<Fid, Error>;
    fn walk(&self, fid: Fid, new_fid: Fid, names: &[&str]) -> Result<(), Error>;
    fn open(&self, fid: Fid, mode: u8) -> Result<(), Error>;
    fn create(&self, parent_fid: Fid, name: &str, perm: u32, mode: u8) -> Result<Fid, Error>;
    fn read(&self, fid: Fid, offset: u64, count: u32) -> Result<Vec<u8>, Error>;
    fn write(&self, fid: Fid, offset: u64, data: &[u8]) -> Result<u32, Error>;
    fn clunk(&self, fid: Fid) -> Result<(), Error>;
    fn stat(&self, fid: Fid) -> Result<Stat, Error>;
}

/// Unicorn Client Interface (Device Management)
pub trait DeviceManager {
    fn register_driver(&self, device_id: u32, device_type: u32) -> Result<u32, Error>;
    fn irq_ack(&self, irq: u32) -> Result<(), Error>;
    fn dma_alloc(&self, size: usize) -> Result<(PhysAddr, VirtAddr), Error>;
    fn dma_free(&self, paddr: PhysAddr) -> Result<(), Error>;
}

/// Rio Client Interface (Wayland Transport)
pub trait WaylandTransport {
    fn connect(&self) -> Result<WaylandConnection, Error>;
    fn dispatch(&self, conn: &WaylandConnection) -> Result<(), Error>;
    fn poll_events(&self, conn: &WaylandConnection) -> Option<Vec<u8>>;
    fn send_capability(&self, conn: &WaylandConnection, object_id: u32, cap: CapPtr) -> Result<(), Error>;
}
```
