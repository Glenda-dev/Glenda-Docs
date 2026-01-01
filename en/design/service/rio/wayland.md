# Rio Wayland Interface Implementation Guide

This document details the steps to implement the Wayland protocol support in Rio, enabling Glenda to run standard Wayland clients.

## 1. Overview

Rio will act as a Wayland Compositor. The implementation will leverage the Rust ecosystem (specifically `wayland-server`) to handle the protocol serialization and state management, while integrating with Glenda's native subsystems (Factotum for memory, Unicorn for input/output).

## 2. Prerequisites

*   **Rust Crates**: `wayland-server`, `wayland-protocols`.
*   **IPC Bridge**: A mechanism to transport Wayland wire protocol messages over Glenda IPC or a Unix Socket provided by Tux.

## 3. Implementation Steps

### Step 1: Project Setup & Dependencies
Add the necessary dependencies to `service/rio/Cargo.toml`.

```toml
[dependencies]
wayland-server = "0.30" # Or latest version
wayland-protocols = { version = "0.30", features = ["client", "server"] }
```

### Step 2: Transport Layer Abstraction
Standard Wayland uses Unix Domain Sockets. In Glenda, we have two options:
1.  **Tux Integration**: Rio asks Tux to listen on `/run/user/X/wayland-0`. Tux forwards the file descriptor or data stream to Rio.
2.  **Native IPC Tunnel**: A custom transport where Wayland messages are wrapped in Glenda IPC messages.

**Action**: Implement a `ClientConnection` struct that implements the `wayland_server::socket::ListeningSocket` trait or manually feeds the `wayland_server::Display` with bytes received from the IPC/Socket.

### Step 3: The Global Registry (`wl_registry`)
Initialize the Wayland Display and advertise the global objects.

1.  Create `Display` and `EventLoop`.
2.  Implement the `GlobalDispatch` trait for the core globals.
3.  Register `wl_compositor`, `wl_shm`, `wl_seat`, `wl_output`.

### Step 4: Compositor & Surface Management (`wl_compositor`)
Implement the `wl_compositor` interface to allow clients to create surfaces.

1.  **Create Surface**: When a client calls `create_surface`, allocate a `RioSurface` struct.
2.  **State Tracking**: `RioSurface` must track:
    *   Pending state (buffer, damage, input region).
    *   Current state (committed).
    *   Double-buffering logic (apply pending to current on `wl_surface.commit`).

### Step 5: Memory & Buffer Management (`wl_shm`)
This is the most critical integration point with Glenda.

1.  **File Descriptor Translation**: Wayland clients send a file descriptor for shared memory via `wl_shm.create_pool`.
    *   *Challenge*: Glenda uses Capabilities, not POSIX FDs natively.
    *   *Solution*: If running via Tux, Tux converts the FD to a Glenda Memory Capability and passes it to Rio. Rio maps this Capability.
2.  **Buffer Access**: Implement `wl_buffer`. When attached to a surface, Rio must be able to read the pixel data from the mapped memory.

### Step 6: Shell Protocol (`xdg_shell`)
Implement `xdg_wm_base` to manage application windows.

1.  **XDG Surface**: Handle the mapping of a `wl_surface` to a desktop window role.
2.  **Toplevels**: Implement `xdg_toplevel` for main application windows (resize, maximize, close events).
3.  **Popups**: Implement `xdg_popup` for menus and tooltips.
4.  **Configure Events**: Send configure events to clients to negotiate window size.

### Step 7: Input Handling (`wl_seat`)
Bridge Rio's native input system (from Unicorn) to Wayland events.

1.  **Capabilities**: Advertise Pointer and Keyboard capabilities.
2.  **Focus Management**: Track which surface has keyboard focus and pointer focus.
3.  **Event Dispatch**:
    *   Convert Glenda Input Events -> Wayland Protocol Events.
    *   `wl_pointer.motion`, `wl_pointer.button`.
    *   `wl_keyboard.key`, `wl_keyboard.enter`, `wl_keyboard.leave`.

### Step 8: Rendering & Output (`wl_output`)
1.  **Advertise Outputs**: Publish `wl_output` globals for each connected screen.
2.  **Frame Callbacks**: Implement `wl_surface.frame`.
    *   Rio must send the `done` event to the callback when the frame is ready to be rendered (usually synced with VSync).
3.  **Compositing Loop**:
    *   Iterate over all visible surfaces.
    *   Read their buffers.
    *   Composite them into the main framebuffer.

## 4. Future Considerations
*   **DMA-BUF Support**: For high-performance zero-copy rendering (`linux-dmabuf-v1`).
*   **Clipboard**: Implement `wl_data_device` for copy-paste support.
*   **Subsurfaces**: For complex UI toolkits.
