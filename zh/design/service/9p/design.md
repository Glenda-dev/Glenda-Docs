# 9P 命名空间服务器设计文档

## 1. 简介
**9P 命名空间服务器** (通常简称为“命名空间服务器”或 `ns`) 是 Glenda 中的中央命名权威。受 Plan 9 启发，它允许每个进程拥有文件系统层次结构的自定义视图。

## 2. 职责

*   **命名空间构建**: 将多个服务组合成单个树。
    *   将 **Fossil** 挂载到 `/`。
    *   将 **Gopher** 绑定到 `/net`。
    *   将 **Unicorn** 设备绑定到 `/dev`。
    *   将 **9Ball/Proc** 绑定到 `/proc`。
*   **路径解析**: 将路径 (`/usr/bin/hello`) 解析为特定的 `{Endpoint, FileID}` 对。
*   **进程隔离**: 不同的进程可以拥有不同的命名空间（例如容器）。

## 3. 架构

### 3.1 清单 (Manifest)
服务器为每个进程组维护一个 **清单 (Manifest)** (挂载表)。
```rust
struct Namespace {
    mounts: BTreeMap<Path, MountEntry>,
}
struct MountEntry {
    server_ep: Endpoint, // 服务的 Capability (例如 Fossil)
    root_fid: u32,       // 该服务上的根文件 ID
    flags: MountFlags,   // Bind Before, Bind After, Create
}
```

### 3.2 解析流程 (客户端缓存)
为了避免成为瓶颈，命名空间服务器主要在跨越挂载点的 `open` 或 `walk` 期间被咨询。
1.  **APE** (客户端库) 缓存已解析的 Capability。
2.  如果路径跨越挂载点，APE 联系 9P Server。
3.  9P Server 返回目标服务（例如 Fossil）的 **Endpoint** 和新的根 FID。
4.  随后的操作 (Read/Write) **直接** 发往 Fossil。命名空间服务器退出数据路径。

## 4. 操作

*   `bind(new, old, flags)`: 使 `new` 在 `old` 处可见。
*   `mount(channel, old, flags)`: 在 `old` 处挂载服务连接。
*   `unmount(old)`: 移除绑定。
*   `impersonate(pid)`: (仅管理员) 调试另一个进程的命名空间。

## 5. 与其他服务的关系

*   **Fossil/Gopher/Unicorn**: 这些是“叶子服务器”。它们导出资源树。
*   **9P Namespace Server**: 这是“根服务器”或“目录服务”。它导出 **结构**。
