# Glenda 协议接口规范

本文档定义了 Glenda 系统中各组件之间通信的协议接口。

## 1. IPC 消息格式

Glenda 的 IPC 消息通过 UTCB (User Thread Control Block) 和寄存器传递。

### 1.1 消息标签 (MsgTag)

每条消息以 `MsgTag` 开头，通常位于寄存器 `MR0`（或架构特定的参数寄存器）中。

```rust
struct MsgTag {
    label: u16,   // 协议标签 (Protocol ID)
    flags: u4,    // 标志位 (例如 HAS_CAP)
    length: u4,   // 消息长度 (寄存器数量)
}
```

### 1.2 协议标签

标签用于区分不同的服务请求或事件类型。

#### 内核保留标签 (Kernel Protocols)
| 标签 | 名称 | 描述 |
| :--- | :--- | :--- |
| `0xFFFF` | `PAGE_FAULT` | 缺页异常通知 |
| `0xFFFE` | `EXCEPTION` | 通用异常通知 |
| `0xFFFD` | `UNKNOWN_SYSCALL` | 未知系统调用 |
| `0xFFFC` | `CAP_FAULT` | Capability 错误 |
| `0xFFFB` | `IRQ` | 硬件中断通知 |
| `0xFFFA` | `NOTIFY` | 异步通知 (仅 Badge) |

#### 系统服务标签 (System Service Protocols)
| 范围 | 服务 | 描述 |
| :--- | :--- | :--- |
| `0x0000` - `0x00FF` | **Generic** | 通用控制协议 (Ping, Debug) |
| `0x0100` - `0x01FF` | **Factotum** | 进程和线程管理 |
| `0x0200` - `0x02FF` | **Gopher** | 文件系统和 IO |
| `0x0300` - `0x03FF` | **Unicorn** | 设备驱动控制 |
| `0x0400` - `0x04FF` | **Rio** | 图形显示协议 |

## 2. 详细协议定义

### 2.1 Factotum 协议 (进程与内存)

Factotum 充当中央进程管理器和异常处理程序。

**基准协议 ID**: `0x0100`

| ID | 方法 | 参数 (UTCB/Regs) | 返回值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| `0x0101` | `SPAWN` | `[name_ptr, name_len]` | `[pid]` | 从文件系统路径生成新进程。 |
| `0x0102` | `EXIT` | `[status]` | - | 终止调用进程。 |
| `0x0103` | `WAIT` | `[pid]` | `[status]` | 阻塞直到指定的子进程退出。 |
| `0x0104` | `YIELD` | `[]` | `[]` | 放弃当前时间片。 |
| `0x0105` | `SBRK` | `[increment]` | `[new_break]` | 增加进程堆大小。 |
| `0x0106` | `MAP_DEVICE` | `[paddr, size, flags]` | `[vaddr]` | 映射物理设备区域（需要特权）。 |
| `0x0107` | `GET_PID` | `[]` | `[pid]` | 返回当前进程 ID。 |

### 2.2 Gopher 协议 (9P2000)

Gopher 实现了标准的 **9P2000** 协议。Glenda IPC 用作传输层，取代了传统的 TCP 或管道传输。这允许网络透明性和统一的资源接口。

**传输机制:**
*   **请求**: 客户端构造标准的 9P `T-message` 并将其写入 **UTCB (IPC Buffer)**。
*   **IPC 调用**: 客户端在 Gopher 的 Endpoint 上调用 `Call`。
    *   **标签**: `0x0200` (9P_REQUEST)
    *   **参数**: `[msg_length, 0, 0, 0, 0, 0]`
*   **响应**: Gopher 处理请求并将相应的 9P `R-message` 写回客户端的 UTCB。

**支持的 9P 操作:**

| 9P 消息类型 | 描述 | Glenda 实现说明 |
| :--- | :--- | :--- |
| `Tversion` / `Rversion` | 协议版本协商 | 协商缓冲区大小 (msize) |
| `Tauth` / `Rauth` | 认证 | 可以使用 Capabilities 进行认证 |
| `Tattach` / `Rattach` | 建立到根的连接 | 返回根 `fid` |
| `Twalk` / `Rwalk` | 遍历目录层次结构 | 逐个组件解析路径 |
| `Topen` / `Ropen` | 打开文件 | 准备 `fid` 用于 I/O |
| `Tcreate` / `Rcreate` | 创建文件 | 在父目录中创建文件 |
| `Tread` / `Rread` | 读取数据 | 数据在 UTCB 中返回（或共享内存） |
| `Twrite` / `Rwrite` | 写入数据 | 数据通过 UTCB 发送（或共享内存） |
| `Tclunk` / `Rclunk` | 关闭 fid | 释放句柄 |
| `Tremove` / `Rremove` | 删除文件 | 删除文件 |
| `Tstat` / `Rstat` | 获取属性 | 检索文件元数据 |
| `Twstat` / `Rwstat` | 设置属性 | 更新文件元数据 |

**大 I/O 优化 (零拷贝):**
由于 UTCB 大小有限（通常为 4KB），标准的 `Tread`/`Twrite` 对于大数据效率低下。
*   **共享内存**: 客户端可以将 Frame Capability（共享内存）传递给 Gopher。
*   **批量 IO**: 可以使用专门的扩展或单独的 `READ_BULK` / `WRITE_BULK` 方法直接向/从共享帧传输数据，绕过消息复制。

### 2.3 Unicorn 协议 (设备驱动)

Unicorn 管理设备发现、中断路由和 DMA 内存分配。

**基准协议 ID**: `0x0300`

| ID | 方法 | 参数 | 返回值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| `0x0301` | `REGISTER` | `[device_id, type]` | `[driver_id]` | 注册新的驱动程序实例。 |
| `0x0302` | `IRQ_ACK` | `[irq_num]` | `[]` | 确认中断已处理。 |
| `0x0303` | `DMA_ALLOC` | `[size]` | `[paddr, vaddr]` | 分配支持 DMA 的连续内存。 |
| `0x0304` | `DMA_FREE` | `[paddr]` | `[]` | 释放 DMA 内存。 |

### 2.4 Rio 协议 (Wayland Compositor)

Rio 实现了一个 **Wayland 兼容** 的合成器。它使用 Glenda IPC 作为 Wayland 线路协议的传输层，取代了 Unix 域套接字。

**传输机制:**
*   **线路协议**: 标准 Wayland 面向对象协议。
*   **共享内存**: 用于命令缓冲区（高吞吐量）和像素缓冲区 (`wl_shm`)。
*   **Capabilities**: 用于传递共享内存句柄（取代文件描述符）。

**基准协议 ID**: `0x0400`

| ID | 方法 | 参数 | 返回值 | 描述 |
| :--- | :--- | :--- | :--- | :--- |
| `0x0401` | `CONNECT` | `[]` | `[conn_id]` | 建立新的 Wayland 客户端连接。 |
| `0x0402` | `DISPATCH` | `[conn_id]` | `[]` | 通知 Rio 处理共享命令缓冲区中的消息。 |
| `0x0403` | `GET_EVENT` | `[conn_id]` | `[has_event]` | 检查/等待来自合成器的事件。 |
| `0x0404` | `PASS_CAP` | `[conn_id, obj_id]` | `[]` | 传输与协议对象关联的 Capability（例如 Frame）。 |

**支持的关键 Wayland 全局对象:**
*   `wl_compositor`: Surface 创建。
*   `wl_shm`: 共享内存缓冲区。
*   `wl_seat`: 输入设备（指针、键盘）。
*   `xdg_wm_base`: 桌面窗口管理。

### 2.5 异常处理协议

当线程发生异常时，内核发送给注册的 Fault Handler（通常是 Factotum）的消息。

**缺页异常 (标签: 0xFFFF)**
*   **发送者**: Kernel
*   **接收者**: Factotum
*   **Payload**:
    *   `Arg0`: `scause` (异常原因)
    *   `Arg1`: `stval` (出错的虚拟地址)
    *   `Arg2`: `sepc` (异常程序计数器)

**通用异常 (标签: 0xFFFE)**
*   **发送者**: Kernel
*   **接收者**: Factotum
*   **Payload**:
    *   `Arg0`: `scause`
    *   `Arg1`: `stval`
    *   `Arg2`: `sepc`

## 3. 进程 Capability 空间 (CSpace) 布局

为了与系统组件通信，进程必须拥有相应的 **Endpoint Capabilities**。当 Factotum 生成新进程时，它会用这些“众所周知的 Capabilities”填充新进程的 CSpace。

| CPtr (索引) | 名称 | 类型 | 描述 |
| :--- | :--- | :--- | :--- |
| `0` | `NULL` | - | 保留 / 无效 |
| `1` | `TCB_SELF` | TCB | 自身 TCB 的 Capability |
| `2` | `CNODE_SELF` | CNode | 自身 CNode 的 Capability |
| `3` | `VSPACE_SELF` | PageTable | 自身 VSpace (根页表) 的 Capability |
| `4` | `EP_FACTOTUM` | Endpoint | **到 Factotum 的 IPC 通道** (进程管理) |
| `5` | `EP_GOPHER` | Endpoint | **到 Gopher 的 IPC 通道** (VFS 根) |
| `6` | `EP_UNICORN` | Endpoint | 到 Unicorn 的 IPC 通道 (可选/仅驱动) |
| `7` | `EP_RIO` | Endpoint | 到 Rio 的 IPC 通道 (可选/仅 GUI 应用) |
| `10` | `FD_STDIN` | Endpoint/File | 标准输入 |
| `11` | `FD_STDOUT` | Endpoint/File | 标准输出 |
| `12` | `FD_STDERR` | Endpoint/File | 标准错误 |

## 4. 接口定义 (Rust Trait 示例)

```rust
/// Factotum 客户端接口 (进程管理)
pub trait ProcessManager {
    fn spawn(&self, name: &str) -> Result<Pid, Error>;
    fn exit(&self, status: usize) -> !;
    fn wait(&self, pid: Pid) -> Result<usize, Error>;
    fn yield_cpu(&self);
    fn sbrk(&self, increment: isize) -> Result<VirtAddr, Error>;
    fn map_device(&self, paddr: PhysAddr, size: usize, flags: usize) -> Result<VirtAddr, Error>;
    fn get_pid(&self) -> Pid;
}

/// Gopher 客户端接口 (9P2000 抽象)
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

/// Unicorn 客户端接口 (设备管理)
pub trait DeviceManager {
    fn register_driver(&self, device_id: u32, device_type: u32) -> Result<u32, Error>;
    fn irq_ack(&self, irq: u32) -> Result<(), Error>;
    fn dma_alloc(&self, size: usize) -> Result<(PhysAddr, VirtAddr), Error>;
    fn dma_free(&self, paddr: PhysAddr) -> Result<(), Error>;
}

/// Rio 客户端接口 (Wayland 传输)
pub trait WaylandTransport {
    fn connect(&self) -> Result<WaylandConnection, Error>;
    fn dispatch(&self, conn: &WaylandConnection) -> Result<(), Error>;
    fn poll_events(&self, conn: &WaylandConnection) -> Option<Vec<u8>>;
    fn send_capability(&self, conn: &WaylandConnection, object_id: u32, cap: CapPtr) -> Result<(), Error>;
}
```
