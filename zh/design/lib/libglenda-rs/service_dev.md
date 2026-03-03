# Glenda Rust 服务开发指南：引用与多线程处理

## 1. 核心挑战

在为 Glenda 开发多线程系统服务时，经常需要在服务的主结构体中持有其他服务的客户端（Client）引用。由于 Glenda 的服务通常是多线程并行的（例如，一个线程接收请求，多个工作线程处理），必须妥善处理 Rust 的并发安全约束（`Send` + `Sync`）。

## 2. 模式：基于 Trait 的服务客户端

### 2.1 抽象客户端接口
服务接口应由 Trait 定义，以便于单元测试（通过 Mock）以及在不同实现间切换。

```rust
pub trait GopherClient: Send + Sync {
    fn fetch_data(&self, path: &str) -> Result<Vec<u8>, Error>;
}
```

### 2.2 线程安全的所有权管理
持有其他客户端引用时，推荐使用 `Arc` (Atomic Reference Counted) 配合 Trait Object：

```rust
pub struct MyService {
    // 使用 Arc 确保在多线程间共享所有权
    // 使用 dyn 动态分发允许持有任何实现了 Trait 的实现类
    gopher: Arc<dyn GopherClient>,
}
```

## 3. 实现细节与 Capability 管理

在 Glenda 中，所有客户端本质上都是对 `CapPtr` (Capability Pointer) 的包装。

- **`CapPtr` 的并发性**：`CapPtr` 通常是一个 `usize` 的透明包装。在微内核中，`CapPtr` 指向进程 CSpace 中的一个槽位。
- **线程局部 UTCB**：发起 IPC 必须使用当前线程的 `UTCB`。
- **克隆与引用**：
  - 如果多个线程共享同一个 `CapPtr` 发起请求，必须确保 IPC 框架支持并发调用（例如通过 `Reply` 槽位区分）。
  - 在 `libglenda-rs` 中，客户端通常是无状态的（Stateless），仅持有 `CapPtr` 并在调用时访问 `TLS (Thread Local Storage)` 获取当前线程的 UTCB。

## 4. 推荐的初始化模式

1. **Warren 分发 Cap**：根进程 Warren 将目标服务的 Endpoint Cap 分发给新创建的服务或通过 `ResourceClient` 查询获取。
2. **构造客户端**：将 `CapPtr` 包装进具体实现类（如 `GopherClientImpl`）。
3. **注入依赖**：利用 `Arc::new(...)` 注入到主服务逻辑中。

```rust
let gopher_cap = resource_client.find_service("gopher")?;
let gopher_impl = Arc::new(GopherClientImpl::new(gopher_cap));

let my_server = MyServer::new(gopher_impl);
// 现在 my_server 可以被 Move 进不同的工作线程
```
