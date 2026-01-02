# Rio 设计文档

## 1. 简介
Rio 是 Glenda 的显示和窗口管理器。它管理图形用户界面，合成来自不同应用程序的窗口，并处理用户输入（键盘、鼠标、触摸）。

## 2. 职责

*   **合成**:
    *   管理窗口的场景图。
    *   将客户端提供的缓冲区合成到最终的帧缓冲区中。
*   **输入路由**:
    *   从 Unicorn（或特定的输入驱动程序）读取输入事件。
    *   将事件分发给获得焦点的窗口/应用程序。
*   **显示管理**:
    *   通过 GPU 驱动程序配置显示分辨率、刷新率和多显示器设置。
*   **协议支持**:
    *   原生 Rio 协议（共享内存 + IPC）。
    *   Wayland 支持（参见 [Wayland 实现](wayland.md)）。

## 3. 架构

Rio 持有物理帧缓冲区（由 Unicorn/GPU 驱动程序提供）的独占 Capability。

### 客户端交互
1.  **缓冲区分配**: 客户端向 Rio 请求共享内存缓冲区（或分配一个并共享它）。
2.  **渲染**: 客户端将内容渲染到共享缓冲区中。
3.  **提交**: 客户端通过 IPC 通知 Rio 帧已准备就绪。
4.  **合成**: Rio 在下一个 VSync 期间将客户端缓冲区位块传输 (blit) 到屏幕。

## 4. 接口

*   **接口**: `org.glenda.ui.Rio`
*   **方法**:
    *   `CreateWindow(width: u32, height: u32) -> WindowHandle`
    *   `GetBuffer(window: WindowHandle) -> ShmCap`
    *   `Commit(window: WindowHandle)`
    *   `PollEvents() -> [Event]`
