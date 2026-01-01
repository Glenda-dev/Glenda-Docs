# Gopher Design Document

## 1. Introduction
Gopher is the Namespace and VFS (Virtual File System) Server for Glenda. It provides a unified hierarchical view of system resources, implementing the **9P2000** protocol. This design is heavily inspired by Plan 9, where "everything is a file".

## 2. Responsibilities

*   **9P2000 Server**:
    *   Implements the server side of the 9P2000 protocol.
    *   Serves the root filesystem (`/`).
    *   Handles 9P messages like `Tattach`, `Twalk`, `Topen`, `Tread`, `Twrite`, `Tclunk`, etc.
*   **Namespace Management**:
    *   Maintains **Per-Process Namespaces** (similar to Plan 9).
    *   Handles `mount` and `bind` operations, allowing processes to customize their own file system view.
*   **VFS Abstraction**:
    *   Abstracts different filesystems (Ext4, FAT, TmpFS) and devices.
    *   Routes 9P requests to the appropriate backend (Driver or FS implementation).
*   **Network Stack**:
    *   Integrates LWIP (Lightweight IP) to provide TCP/IP networking.
    *   Exposes network interfaces as files (e.g., `/net/tcp`, `/net/udp`) via the 9P interface.
*   **Synthetic Filesystems**:
    *   Provides `/proc`, `/dev`, `/sys` like interfaces.
    *   Implements Unix Pipes as synthetic files.
*   **FUSE Support**:
    *   Implements the FUSE kernel protocol to support `libfuse` based filesystems.

## 3. Architecture

Gopher acts as a central **9P Multiplexer**.

### 3.1 9P Transport over IPC
Gopher uses Glenda's IPC mechanism as the transport layer for 9P messages.
*   **Request**: Clients write serialized 9P `T-messages` into the IPC buffer (UTCB) and invoke the Gopher Endpoint.
*   **Response**: Gopher processes the request and writes back `R-messages`.
*   **Zero-Copy**: For large reads/writes, Gopher supports passing Shared Memory Capabilities (Frames) to avoid copying data through the kernel.

### 3.2 Backend Adapters
Gopher translates 9P operations into backend-specific actions:
*   **Block Storage**: Communicates with Block Drivers (via Unicorn) to read/write raw disk sectors for physical filesystems (Ext2/4).
*   **Network**: Translates file operations on `/net/tcp/clone` etc., into LWIP socket calls.
*   **Devices**: Forwards requests to device drivers registered by Unicorn.

## 4. Protocol Interface (9P2000)

Gopher does not use ad-hoc IPC methods. Instead, it strictly follows the 9P2000 protocol specification.

### Supported 9P Messages

| Message Type | Function | Description |
| :--- | :--- | :--- |
| `Tversion` / `Rversion` | Version Negotiation | Negotiates protocol version ("9P2000") and message size (`msize`). |
| `Tauth` / `Rauth` | Authentication | (Optional) Authenticates the connection. |
| `Tattach` / `Rattach` | Root Connection | Establishes a connection to the file tree root, returning a root `fid`. |
| `Twalk` / `Rwalk` | Path Traversal | Descends a directory hierarchy. |
| `Topen` / `Ropen` | Open File | Prepares a `fid` for I/O. |
| `Tcreate` / `Rcreate` | Create File | Creates a new file in a directory. |
| `Tread` / `Rread` | Read Data | Reads data from a file. |
| `Twrite` / `Rwrite` | Write Data | Writes data to a file. |
| `Tclunk` / `Rclunk` | Close Fid | Forgets a `fid` (close). |
| `Tremove` / `Rremove` | Remove File | Removes a file from the server. |
| `Tstat` / `Rstat` | Get Attributes | Reads file metadata (stat). |
| `Twstat` / `Rwstat` | Set Attributes | Writes file metadata. |

## 5. Unix Pipe Implementation

Unix Pipes are implemented as synthetic files managed by **Gopher**, leveraging its 9P2000 capabilities.

### 5.1 Creation Flow
1.  **Request**: The client (via LibC) sends a `Twalk` to find `/dev/pipe` (or similar) and a `Tcreate` (or `Topen` on a clone device) to create a new pipe instance.
2.  **Allocation**: Gopher allocates a kernel-less **Ring Buffer** in its own memory space.
3.  **Handles**: Gopher returns two 9P FIDs (File IDs) pointing to this buffer object.
    *   **Read End**: Opened with `O_READ`.
    *   **Write End**: Opened with `O_WRITE`.

### 5.2 Data Flow
*   **Write**: The writer sends a `Twrite` message. Gopher copies data to the Ring Buffer.
    *   *Blocking*: If full, Gopher delays the `Rwrite` response.
*   **Read**: The reader sends a `Tread` message. Gopher copies data from the Ring Buffer.
    *   *Blocking*: If empty, Gopher delays the `Rread` response.

## 6. FUSE Support

Gopher supports the **FUSE (Filesystem in Userspace)** protocol to allow running existing Linux FUSE filesystems on Glenda.

### 6.1 Architecture
*   **Gopher as Kernel**: Gopher acts as the "kernel" side of the FUSE protocol.
*   **Device Node**: Gopher exposes a synthetic device file `/dev/fuse`.
*   **Translation**: Gopher translates incoming **9P2000** requests into **FUSE** protocol messages.

### 6.2 Workflow
1.  **Mount**: A FUSE daemon (linked with `libfuse`) opens `/dev/fuse` and sends a mount request.
2.  **Request**: When a client accesses a file in the FUSE mount point, Gopher receives a 9P message (e.g., `Tread`).
3.  **Translation**: Gopher converts `Tread` into a `FUSE_READ` opcode.
4.  **Dispatch**: Gopher writes the `FUSE_READ` structure into the buffer associated with the `/dev/fuse` handle.
5.  **Process**: The FUSE daemon reads from `/dev/fuse`, processes the request, and writes a reply.
6.  **Reply**: Gopher reads the reply, converts it back to a 9P `Rread`, and sends it to the client.
## 7. Backend Filesystem Protocol

To ensure modularity, complex filesystems (like Ext4, FAT32, NTFS) run as separate user-space processes (FS Services). Gopher connects to them using the standard **9P2000** protocol.

### 7.1 Protocol Standard
*   **Protocol**: 9P2000.
*   **Role**: The FS Service acts as a **9P Server**. Gopher acts as a **9P Client**.
*   **Transport**: Glenda IPC.

### 7.2 Connection Lifecycle
1.  **Startup**: The FS Service (e.g., `fs-ext4`) is started.
2.  **Handshake**: The FS Service provides its Endpoint Capability to Gopher (via 9Ball configuration or a `mount` syscall passing the capability).
3.  **Negotiation**: Gopher sends `Tversion` to the FS Service to negotiate message size (`msize`).
4.  **Attachment**: Gopher sends `Tattach` to obtain the root FID of the external filesystem.
5.  **Binding**: Gopher binds this root FID to a mount point in the global namespace (e.g., `/mnt/data`).

### 7.3 Block Device Access
FS Services typically need to access physical storage.
*   **Direct Access**: The FS Service obtains a Block Device Capability (from Unicorn).
*   **Bypass Gopher**: The FS Service talks directly to the Block Driver for sector I/O. Gopher only handles the high-level file 9P traffic.

### 7.4 Control Protocol (Mounting)
To mount an external FS Service, a client sends a specific `Twrite` to Gopher's control file (e.g., `/sys/mount`).

*   **Command**: `mount <mountpoint> <flags> <service_cap_cptr>`
*   **Action**: Gopher looks up the capability in the client's CSpace, copies it, and initiates the 9P connection described in 7.2.

## 8. Per-Process Namespaces

Gopher implements Plan 9-style per-process namespaces, allowing each process (or group of processes) to have a unique view of the file system.

### 8.1 Namespace Groups
*   **Namespace ID**: Gopher assigns a Namespace ID to each client connection (identified by the Badge of the Endpoint Capability).
*   **Sharing**: By default, a child process shares the namespace of its parent (inherits the same Namespace ID).
*   **Isolation**: A process can request to detach from the shared namespace and create a private copy.

### 8.2 Operations
*   **Clone Namespace (RFORK_NS)**:
    *   Triggered when Factotum spawns a process with the `RFNOMNT` flag.
    *   Gopher copies the current mount table to a new Namespace ID.
    *   Subsequent `mount`/`bind` operations only affect the new namespace.
*   **Mount/Bind**:
    *   Modifies the current namespace view.
    *   Example: `bind /tmp/private /tmp` overlays a private directory on `/tmp` only for the current process.

### 8.3 Implementation
*   **Mount Table**: Gopher maintains a map: `Map<NamespaceID, MountTable>`.
*   **Lookup**: When resolving paths, Gopher first looks up the `MountTable` associated with the client's Namespace ID.

