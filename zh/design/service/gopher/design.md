# Gopher 设计文档

## 1. 简介
Gopher 是 Glenda 的命名空间和 VFS (虚拟文件系统) 服务器。它提供系统资源的统一分层视图，实现了 **9P2000** 协议。该设计深受 Plan 9 的启发，即“一切皆文件”。

## 2. 职责

*   **9P2000 服务器**:
    *   实现 9P2000 协议的服务器端。
    *   提供根文件系统 (`/`) 服务。
    *   处理 9P 消息，如 `Tattach`, `Twalk`, `Topen`, `Tread`, `Twrite`, `Tclunk` 等。
*   **命名空间管理**:
    *   维护 **逐进程命名空间** (类似于 Plan 9)。
    *   处理 `mount` 和 `bind` 操作，允许进程自定义其文件系统视图。
*   **VFS 抽象**:
    *   抽象不同的文件系统 (Ext4, FAT, TmpFS) 和设备。
    *   将 9P 请求路由到相应的后端（驱动程序或 FS 实现）。
*   **网络栈**:
    *   集成 LWIP (Lightweight IP) 以提供 TCP/IP 网络。
    *   通过 9P 接口将网络接口暴露为文件（例如 `/net/tcp`, `/net/udp`）。
*   **合成文件系统**:
    *   提供类似 `/proc`, `/dev`, `/sys` 的接口。
    *   将 Unix 管道实现为合成文件。
*   **FUSE 支持**:
    *   实现 FUSE 内核协议以支持基于 `libfuse` 的文件系统。

## 3. 架构

Gopher 充当中央 **9P 多路复用器**。

### 3.1 基于 IPC 的 9P 传输
Gopher 使用 Glenda 的 IPC 机制作为 9P 消息的传输层。
*   **请求**: 客户端将序列化的 9P `T-message` 写入 IPC 缓冲区 (UTCB) 并调用 Gopher Endpoint。
*   **响应**: Gopher 处理请求并将 `R-message` 写回。
*   **零拷贝**: 对于大读/写，Gopher 支持传递共享内存 Capabilities (Frames) 以避免通过内核复制数据。

### 3.2 后端适配器
Gopher 将 9P 操作转换为特定于后端的动作：
*   **块存储**: 与块驱动程序（通过 Unicorn）通信，为物理文件系统 (Ext2/4) 读/写原始磁盘扇区。
*   **网络**: 将 `/net/tcp/clone` 等上的文件操作转换为 LWIP 套接字调用。
*   **设备**: 将请求转发给 Unicorn 注册的设备驱动程序。

## 4. 协议接口 (9P2000)

Gopher 不使用临时的 IPC 方法。相反，它严格遵循 9P2000 协议规范。

### 支持的 9P 消息

| 消息类型 | 功能 | 描述 |
| :--- | :--- | :--- |
| `Tversion` / `Rversion` | 版本协商 | 协商协议版本 ("9P2000") 和消息大小 (`msize`)。 |
| `Tauth` / `Rauth` | 认证 | (可选) 认证连接。 |
| `Tattach` / `Rattach` | 根连接 | 建立到文件树根的连接，返回根 `fid`。 |
| `Twalk` / `Rwalk` | 路径遍历 | 下降目录层次结构。 |
| `Topen` / `Ropen` | 打开文件 | 准备 `fid` 用于 I/O。 |
| `Tcreate` / `Rcreate` | 创建文件 | 在目录中创建新文件。 |
| `Tread` / `Rread` | 读取数据 | 从文件读取数据。 |
| `Twrite` / `Rwrite` | 写入数据 | 向文件写入数据。 |
| `Tclunk` / `Rclunk` | 关闭 Fid | 忘记 `fid` (关闭)。 |
| `Tremove` / `Rremove` | 删除文件 | 从服务器删除文件。 |
| `Tstat` / `Rstat` | 获取属性 | 读取文件元数据 (stat)。 |
| `Twstat` / `Rwstat` | 设置属性 | 写入文件元数据。 |

## 5. Unix 管道实现

Unix 管道作为由 **Gopher** 管理的合成文件实现，利用其 9P2000 能力。

### 5.1 创建流程
1.  **请求**: 客户端（通过 LibC）发送 `Twalk` 查找 `/dev/pipe`（或类似物），并发送 `Tcreate`（或在克隆设备上发送 `Topen`）以创建新的管道实例。
2.  **分配**: Gopher 在其自己的内存空间中分配一个无内核的 **环形缓冲区**。
3.  **句柄**: Gopher 返回指向此缓冲区对象的两个 9P FID (文件 ID)。
    *   **读端**: 以 `O_READ` 打开。
    *   **写端**: 以 `O_WRITE` 打开。

### 5.2 数据流
*   **写**: 写者发送 `Twrite` 消息。Gopher 将数据复制到环形缓冲区。
    *   *阻塞*: 如果已满，Gopher 延迟 `Rwrite` 响应。
*   **读**: 读者发送 `Tread` 消息。Gopher 从环形缓冲区复制数据。
    *   *阻塞*: 如果为空，Gopher 延迟 `Rread` 响应。

## 6. FUSE 支持

Gopher 支持 **FUSE (用户空间文件系统)** 协议，允许在 Glenda 上运行现有的 Linux FUSE 文件系统。

### 6.1 架构
*   **Gopher 作为内核**: Gopher 充当 FUSE 协议的“内核”侧。
*   **设备节点**: Gopher 暴露一个合成设备文件 `/dev/fuse`。
*   **转换**: Gopher 将传入的 **9P2000** 请求转换为 **FUSE** 协议消息。

### 6.2 工作流
1.  **挂载**: FUSE 守护进程（链接了 `libfuse`）打开 `/dev/fuse` 并发送挂载请求。
2.  **请求**: 当客户端访问 FUSE 挂载点中的文件时，Gopher 接收 9P 消息（例如 `Tread`）。
3.  **转换**: Gopher 将 `Tread` 转换为 `FUSE_READ` 操作码。
4.  **分发**: Gopher 将 `FUSE_READ` 结构写入与 `/dev/fuse` 句柄关联的缓冲区。
5.  **处理**: FUSE 守护进程从 `/dev/fuse` 读取，处理请求，并写入回复。
6.  **回复**: Gopher 读取回复，将其转换回 9P `Rread`，并发送给客户端。

## 7. 后端文件系统协议

为了确保模块化，复杂的文件系统（如 Ext4, FAT32, NTFS）作为单独的用户空间进程 (FS Service) 运行。Gopher 使用标准的 **9P2000** 协议连接到它们。

### 7.1 协议标准
*   **协议**: 9P2000。
*   **角色**: FS Service 充当 **9P Server**。Gopher 充当 **9P Client**。
*   **传输**: Glenda IPC。

### 7.2 连接生命周期
1.  **启动**: FS Service (例如 `fs-ext4`) 启动。
2.  **握手**: FS Service 向 Gopher 提供其 Endpoint Capability（通过 9Ball 配置或传递 capability 的 `mount` 系统调用）。
3.  **协商**: Gopher 向 FS Service 发送 `Tversion` 以协商消息大小 (`msize`)。
4.  **连接**: Gopher 发送 `Tattach` 以获取外部文件系统的根 FID。
5.  **绑定**: Gopher 将此根 FID 绑定到全局命名空间中的挂载点（例如 `/mnt/data`）。

### 7.3 块设备访问
FS Service 通常需要访问物理存储。
*   **直接访问**: FS Service 获取块设备 Capability（来自 Unicorn）。
*   **绕过 Gopher**: FS Service 直接与块驱动程序对话以进行扇区 I/O。Gopher 仅处理高级文件 9P 流量。

### 7.4 控制协议 (挂载)
要挂载外部 FS Service，客户端向 Gopher 的控制文件（例如 `/sys/mount`）发送特定的 `Twrite`。

*   **命令**: `mount <mountpoint> <flags> <service_cap_cptr>`
*   **动作**: Gopher 在客户端的 CSpace 中查找 capability，复制它，并启动 7.2 中描述的 9P 连接。

## 8. 逐进程命名空间

Gopher 实现了 Plan 9 风格的逐进程命名空间，允许每个进程（或进程组）拥有唯一的文件系统视图。

### 8.1 命名空间组
*   **命名空间 ID**: Gopher 为每个客户端连接（由 Endpoint Capability 的 Badge 标识）分配一个命名空间 ID。
*   **共享**: 默认情况下，子进程共享其父进程的命名空间（继承相同的命名空间 ID）。
*   **隔离**: 进程可以请求脱离共享命名空间并创建私有副本。

### 8.2 操作
*   **克隆命名空间 (RFORK_NS)**:
    *   当 Factotum 使用 `RFNOMNT` 标志生成进程时触发。
    *   Gopher 将当前挂载表复制到新的命名空间 ID。
    *   后续的 `mount`/`bind` 操作仅影响新命名空间。
*   **Mount/Bind**:
    *   修改当前命名空间视图。
    *   示例：`bind /tmp/private /tmp` 仅为当前进程将私有目录覆盖在 `/tmp` 上。

### 8.3 实现
*   **挂载表**: Gopher 维护一个映射：`Map<NamespaceID, MountTable>`。
*   **查找**: 解析路径时，Gopher 首先查找与客户端命名空间 ID 关联的 `MountTable`。
