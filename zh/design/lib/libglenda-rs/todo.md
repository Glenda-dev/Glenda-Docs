# libglenda-rs 待办事项

## 核心功能
- [ ] **系统调用包装器**: 实现所有内核系统调用的 Rust 包装器。
- [ ] **UTCB 访问**: 提供安全访问 UTCB 字段的机制。
- [ ] **Capability 管理**: 实现 `CNode` 操作（mint, copy, move, delete, revoke）。
- [ ] **IPC 接口**: 实现 `send`, `recv`, `call`, `reply` 等 IPC 原语。
- [ ] **内存管理**: 实现 `VSpace` 和 `PageTable` 操作。
- [ ] **线程管理**: 实现 `TCB` 操作（configure, resume, suspend）。

## C 兼容性
- [ ] **C API 导出**: 使用 `extern "C"` 导出核心功能。
- [ ] **头文件生成**: 配置 `cbindgen` 生成 `glenda.h`。
- [ ] **C 运行时 (crt0)**: 实现 C 程序的启动代码。

## 高级抽象
- [ ] **Allocator**: 实现全局分配器 (`#[global_allocator]`)。
- [ ] **Synchronization**: 实现基本的同步原语（如 Mutex, Condvar）。
- [ ] **Async Runtime**: (可选) 实现基本的异步运行时支持。

## 测试与文档
- [ ] **单元测试**: 编写核心功能的单元测试。
- [ ] **集成测试**: 编写与内核交互的集成测试。
- [ ] **API 文档**: 编写详细的 API 文档。
