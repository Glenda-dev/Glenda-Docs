# 9P Namespace Server Design Document

## 1. Introduction
The **9P Namespace Server** (often simply "The Namespace Server" or `ns`) is the central naming authority in Glenda. Inspired by Plan 9, it allows each process to have a customized view of the file system hierarchy.

## 2. Responsibilities

*   **Namespace Construction**: combining multiple services into a single tree.
    *   Mount **Fossil** at `/`.
    *   Bind **Gopher** to `/net`.
    *   Bind **Unicorn** devices to `/dev`.
    *   Bind **9Ball/Proc** to `/proc`.
*   **Path Resolution**: Resolving paths (`/usr/bin/hello`) to specific `{Endpoint, FileID}` pairs.
*   **Process Isolation**: Different processes can have different namespaces (e.g., containers).

## 3. Architecture

### 3.1 The Manifest
The server maintains a **Manifest** (Mount Table) for every process group.
```rust
struct Namespace {
    mounts: BTreeMap<Path, MountEntry>,
}
struct MountEntry {
    server_ep: Endpoint, // Capability to the service (e.g., Fossil)
    root_fid: u32,       // Root file ID on that server
    flags: MountFlags,   // Bind Before, Bind After, Create
}
```

### 3.2 Resolution Flow (Client-Side Caching)
To avoid being a bottleneck, the Namespace Server is mainly consulted during `open` or `walk` spanning mount points.
1.  **APE** (Client Library) caches resolved capabilities.
2.  If a path crosses a mount point, APE contacts 9P Server.
3.  9P Server returns the **Endpoint** of the target service (e.g., Fossil) and the new root FID.
4.  Subsequent operations (Read/Write) go **Directly** to Fossil. The Namespace Server steps out of the data path.

## 4. Operations

*   `bind(new, old, flags)`: Make `new` visible at `old`.
*   `mount(channel, old, flags)`: Mount a service connection at `old`.
*   `unmount(old)`: Remove a binding.
*   `impersonate(pid)`: (Admin only) Debug another process's namespace.

## 5. Relationship with Other Services

*   **Fossil/Gopher/Unicorn**: These are "Leaf Servers". They export resource trees.
*   **9P Namespace Server**: This is the "Root Server" or "Directory Service". It exports the *structure*.
