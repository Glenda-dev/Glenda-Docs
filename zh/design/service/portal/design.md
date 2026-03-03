# Glenda 远程 IPC 协议设计 (Portal 服务)

## 1. 设计背景与理念

在 Glenda 的微内核架构中，所有通信均基于本地 Capability (能力) 和消息传递 (IPC)。为了支持跨节点的分布式通信，需要引入 **Portal** 服务作为 IPC 消息的代理。

Portal 的核心目标是**使远程通信对用户态进程尽可能透明**。它将本地的 IPC 调用序列化，通过底层传输层（Ethernet、RDMA、USB 等）发送到远程节点，并在远端重建 IPC 消息进行投递。

## 2. 架构设计

Portal 服务采用 **代理 Endpoint (Proxy Endpoint)** 模式：

1. **本地代理**：当进程请求远程服务时，其实际持有的是本地 Portal 提供的 Proxy Endpoint。
2. **消息拦截**：进程像往常一样向该代理 Endpoint 发送同步的 `Call` 或 `Send` 消息。
3. **封包与传输**：
   - **Sender 端**：Portal 捕获发送至 Proxy Endpoint 的 IPC，提取 UTCB 中的消息、寄存器和内存传递信息，将其打包、压缩，并通过网络协议栈发送。
   - **Receiver 端**：远端 Portal 接收数据包，根据 Global ID 找到或创建对应的本地真实 Endpoint，重建 UTCB 环境并触发本地 IPC。

## 3. 跨节点 Capability 传递 (DistCap)

Capability 在跨节点传递时，采用 **导出 (Export) / 导入 (Import)** 机制：

- **Global ID (GID)**：每个可跨节点访问的 Capability 分配一个 128 位的全球唯一 ID (GCap ID)。
- **所有权跟踪**：Portal 维护本地 CapPtr 与 GID 的映射表。
- **传递策略**：
  - 发送端将 CapPtr 转换为 GID 发送。Portal 在自己的 CNode 中保存引用以防止销毁。
  - 接收端 Portal 检查 GID 是否已存在本地映射；若无，则向源节点请求 Capability 副本或创建一个 Proxy 句柄（代理端点）。

## 4. 协议栈设计

### 4.1 消息格式 (RIPC Wire Format)

**基准协议 ID**: `0x0500` (Portal)

```rust
struct RipcHeader {
    magic: u32,           // 协议魔数 "GLND"
    session_id: u32,      // 会话 ID，关联 Call 和 Reply
    src_node: u16,        // 源节点 ID
    dst_node: u16,        // 目标节点 ID
    compression_type: u8, // 压缩算法: 0=None, 1=LZ4, 2=ZSTD
    flags: u8,            // 标志位 (IS_REPLY, HAS_CAP_PAYLOAD)
    payload_len: u32,     // 压缩后的 Payload 长度
}

struct RipcPayload {
    msg_tag: u64,         // 原始的 MsgTag
    badge: u64,           // 发送者的身份标识
    mrs_regs: [u64; 7],   // 消息寄存器数据 (MR1-MR7)
    cap_exports: Vec<u64>,// 被传输 Capability 的全球 ID (GCap ID) 列表
    ipc_buffer: Vec<u8>,  // IPC 缓冲区内容（可选压缩）
}
```

### 4.2 传输层抽象 (Transport Abstraction)

Portal 采用插件化架构支持多种物理介质：

- **Ethernet (TCP/UDP)**: 通用传输，对 IPC 缓冲区执行 LZ4/ZSTD 压缩。
- **RDMA (RoCE/InfiniBand)**: 针对大 I/O 零拷贝优化。直接通过内存注册 (MR) 实现跨机器物理页映射。
- **USB4 / Thunderbolt (PCIe Tunneling)**: 
  - 利用 Host-to-Host (XDomain) 特性。
  - 基于 **NHI (Native Host Interface)** 的 DMA 描述符环进行传输。
  - 结合 IOMMU 强制隔离，确保 DMA 访问安全。
- **USB OTG (Device Mode)**: 
  - 嵌入式设备作为外设，暴露 Bulk IN/OUT 端点。
  - 实现逻辑：`服务 -> UTCB -> Portal -> UDC 驱动 -> 硬件发送`。支持 Capability 零拷贝直传。
- **UART / SPI (Slow Path)**:
  - 针对资源受限节点。
  - 引入 **SLIP** 帧定界与 **CRC16** 校验。
  - **短消息优化**：对于无 Buffer 的 Notify 消息，压缩至 8-12 字节以适应低带宽。

## 5. 高级特性

### 5.1 分布式共享内存 (DSM)
利用 DMA 传输层将 `Frame` Capability 跨节点映射。远端节点将特定的物理地址范围包装成本地 `Frame`，实现硬件级的跨节点内存读写。

### 5.2 进程与线程的分布式扩展
- **跨节点实时迁移**：通过 DMA 批量迁移 `CNode` (能力空间) 和 `VSpace` (地址空间)，结合缺页异常 (Page Fault) 实现按需预取。
- **分布式多节点单进程**：使一个进程的线程分布在不同刀片上，共享同一个 VSpace 和 CSpace。
- **刀片服务器集群适配**：将整个机架背板视为统一的交换网格，实现资源解构 (Disaggregation)。

## 6. 跨平台互联 (Hosted Glenda)
通过移植 `libglenda-rs` 并在 Linux/Windows 上实现 `PortalBackend`，使非原生系统能够通过 Portal 协议调用远端 Glenda 集群的资源，实现“Hosted Glenda”运行时。

