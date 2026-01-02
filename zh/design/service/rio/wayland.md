# Rio Wayland 接口实现指南

本文档详细介绍了在 Rio 中实现 Wayland 协议支持的步骤，使 Glenda 能够运行标准的 Wayland 客户端。

## 1. 概述

Rio 将充当 Wayland 合成器。该实现将利用 Rust 生态系统（特别是 `wayland-server`）来处理协议序列化和状态管理，同时与 Glenda 的原生子系统（用于内存的 Factotum，用于输入/输出的 Unicorn）集成。

## 2. 先决条件

*   **Rust Crates**: `wayland-server`, `wayland-protocols`.
*   **IPC 桥接**: 一种通过 Glenda IPC 或 Tux 提供的 Unix 套接字传输 Wayland 线路协议消息的机制。

## 3. 实现步骤

### 步骤 1: 项目设置与依赖
将必要的依赖项添加到 `service/rio/Cargo.toml`。

```toml
[dependencies]
wayland-server = "0.30" # 或最新版本
wayland-protocols = { version = "0.30", features = ["client", "server"] }
```

### 步骤 2: 传输层抽象
标准 Wayland 使用 Unix 域套接字。在 Glenda 中，我们有两个选择：
1.  **Tux 集成**: Rio 请求 Tux 在 `/run/user/X/wayland-0` 上监听。Tux 将文件描述符或数据流转发给 Rio。
2.  **原生 IPC 隧道**: 一种自定义传输，其中 Wayland 消息被包装在 Glenda IPC 消息中。

**行动**: 实现一个 `ClientConnection` 结构体，该结构体实现 `wayland_server::socket::ListeningSocket` trait，或者手动将从 IPC/Socket 接收到的字节馈送到 `wayland_server::Display`。

### 步骤 3: 全局注册表 (`wl_registry`)
初始化 Wayland Display 并广播全局对象。

1.  创建 `Display` 和 `EventLoop`。
2.  为核心全局对象实现 `GlobalDispatch` trait。
3.  注册 `wl_compositor`, `wl_shm`, `wl_seat`, `wl_output`。

### 步骤 4: 合成器与表面管理 (`wl_compositor`)
实现 `wl_compositor` 接口以允许客户端创建表面 (surfaces)。

1.  **创建表面**: 当客户端调用 `create_surface` 时，分配一个 `RioSurface` 结构体。
2.  **状态跟踪**: `RioSurface` 必须跟踪：
    *   挂起状态（缓冲区、损伤、输入区域）。
    *   当前状态（已提交）。
    *   双缓冲逻辑（在 `wl_surface.commit` 时将挂起状态应用到当前状态）。

### 步骤 5: 内存与缓冲区管理 (`wl_shm`)
这是与 Glenda 最关键的集成点。

1.  **文件描述符转换**: Wayland 客户端通过 `wl_shm.create_pool` 发送共享内存的文件描述符。
    *   *挑战*: Glenda 原生使用 Capabilities，而不是 POSIX FD。
    *   *解决方案*: 如果通过 Tux 运行，Tux 将 FD 转换为 Glenda 内存 Capability 并将其传递给 Rio。Rio 映射此 Capability。
2.  **缓冲区访问**: 实现 `wl_buffer`。当附加到表面时，Rio 必须能够从映射的内存中读取像素数据。

### 步骤 6: Shell 协议 (`xdg_shell`)
实现 `xdg_wm_base` 以管理应用程序窗口。

1.  **XDG Surface**: 处理 `wl_surface` 到桌面窗口角色的映射。
2.  **Toplevels**: 为主应用程序窗口实现 `xdg_toplevel`（调整大小、最大化、关闭事件）。
3.  **Popups**: 为菜单和工具提示实现 `xdg_popup`。
4.  **配置事件**: 向客户端发送配置事件以协商窗口大小。

### 步骤 7: 输入处理 (`wl_seat`)
将 Rio 的原生输入系统（来自 Unicorn）桥接到 Wayland 事件。

1.  **Capabilities**: 广播指针和键盘 Capabilities。
2.  **焦点管理**: 跟踪哪个表面具有键盘焦点和指针焦点。
3.  **事件分发**:
    *   转换 Glenda 输入事件 -> Wayland 协议事件。
    *   `wl_pointer.motion`, `wl_pointer.button`.
    *   `wl_keyboard.key`, `wl_keyboard.enter`, `wl_keyboard.leave`.

### 步骤 8: 渲染与输出 (`wl_output`)
1.  **广播输出**: 为每个连接的屏幕发布 `wl_output` 全局对象。
2.  **帧回调**: 实现 `wl_surface.frame`。
    *   当帧准备好渲染时（通常与 VSync 同步），Rio 必须向回调发送 `done` 事件。
3.  **合成循环**:
    *   迭代所有可见表面。
    *   读取它们的缓冲区。
    *   将它们合成到主帧缓冲区中。

## 4. 未来考虑
*   **DMA-BUF 支持**: 用于高性能零拷贝渲染 (`linux-dmabuf-v1`)。
*   **剪贴板**: 实现 `wl_data_device` 以支持复制粘贴。
*   **子表面**: 用于复杂的 UI 工具包。
