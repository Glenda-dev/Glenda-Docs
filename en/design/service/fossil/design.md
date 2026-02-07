# Fossil Design Document

## 1. Introduction
Fossil is the main **File System Server** for Glenda. It manages persistent storage on block devices and exposes a hierarchical file structure via the **9P2000** protocol.

## 2. Architecture

### 2.1 Backend Abstraction
Fossil supports multiple filesystem implementations via a modular backend trait.
*   **Disk Drivers**: Connects to **Unicorn** (Block Device Manager) to read/write sectors.
*   **Supported Formats**:
    *   **FAT32**: For EFI System Partitions and interoperability.
    *   **Ext4** (Read-only initially): Linux compatibility.
    *   **GlendaFS** (Planned): A log-structured filesystem optimized for capability store.

### 2.2 Metadata vs Data
*   **Control Plane (9P)**: Directory traversal (`walk`), file creation (`create`), and status checks (`stat`) are handled via standard 9P messages.
*   **Data Plane (Shared Memory)**:
    *   Large reads/writes utilize a **DMA-like shared memory mechanism**.
    *   Fossil exposes a "Dataport" capability upon file open.
    *   Data is copied directly from File Cache to Client Buffer without intermediate marshalling.

### 2.3 Caching
Fossil integrates with **Warren**'s memory manager.
*   **Page Cache**: Uses a unified page cache where clean pages can be evicted by the kernel/Warren under memory pressure.

## 3. Interfaces

### 3.1 9P Export
Fossil typically serves the root `/` or mount points like `/mnt/disk`.
*   Standard 9P2000.L support.

### 3.2 Block Interface (Client of Unicorn)
Fossil consumes block devices provided by Unicorn.
*   Protocol: `READ_BLOCKS`, `WRITE_BLOCKS` (Async IPC).

## 4. Comparison with Gopher
Previously, Gopher handled VFS duties. Now:
*   **Fossil**: Handles *Disk* filesystems.
*   **Gopher**: Handles *Network* stacks (`/net`).
*   **9P Server**: Aggregates them into a single tree.
