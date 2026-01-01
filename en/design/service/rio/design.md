# Rio Design Document

## 1. Introduction
Rio is the Display and Window Manager for Glenda. It manages the graphical user interface, compositing windows from different applications, and handling user input (keyboard, mouse, touch).

## 2. Responsibilities

*   **Compositing**:
    *   Manages the scene graph of windows.
    *   Composites buffers provided by clients into the final framebuffer.
*   **Input Routing**:
    *   Reads input events from Unicorn (or specific input drivers).
    *   Dispatches events to the focused window/application.
*   **Display Management**:
    *   Configures display resolution, refresh rates, and multi-monitor setups via the GPU driver.
*   **Protocol Support**:
    *   Native Rio Protocol (Shared Memory + IPC).
    *   Wayland Support (See [Wayland Implementation](wayland.md)).

## 3. Architecture

Rio holds the exclusive capability to the physical Framebuffer (provided by Unicorn/GPU Driver).

### Client Interaction
1.  **Buffer Allocation**: Clients request a shared memory buffer from Rio (or allocate one and share it).
2.  **Rendering**: Clients render content into the shared buffer.
3.  **Commit**: Clients notify Rio via IPC that a frame is ready.
4.  **Composite**: Rio blits the client buffer to the screen during the next VSync.

## 4. Interfaces

*   **Interface**: `org.glenda.ui.Rio`
*   **Methods**:
    *   `CreateWindow(width: u32, height: u32) -> WindowHandle`
    *   `GetBuffer(window: WindowHandle) -> ShmCap`
    *   `Commit(window: WindowHandle)`
    *   `PollEvents() -> [Event]`
