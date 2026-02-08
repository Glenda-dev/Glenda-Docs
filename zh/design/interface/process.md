# 进程接口 (Process Interfaces)

## 简介
定义了用于进程生命周期管理、内存控制和异常处理的 Trait。

## ProcessService (`interface/process.rs`)
高级进程控制方法。

```rust
pub trait ProcessService {
    /// 通过名称生成新进程。
    fn spawn(&mut self, name: &str) -> Result<usize, Error>;
    
    /// Fork 当前进程。
    fn fork(&mut self, pid: Badge) -> Result<usize, Error>;
    
    /// 终止进程。
    fn exit(&mut self, pid: Badge, code: usize) -> Result<(), Error>;
    
    /// 将 ELF 二进制镜像加载到目标进程的地址空间。
    fn exec(&mut self, pid: Badge, elf_data: &[u8]) -> Result<(usize, usize), Error>;
}
```

## MemoryService (`interface/memory.rs`)
针对进程的系统级内存操作。

```rust
pub trait MemoryService {
    /// 调整数据段大小 (堆)。
    fn brk(&mut self, pid: Badge, incr: isize) -> Result<usize, Error>;
    
    /// 将内存映射到地址空间。
    fn mmap(&mut self, pid: Badge, addr: usize, len: usize) -> Result<usize, Error>;
    
    /// 从地址空间解除内存映射。
    fn munmap(&mut self, pid: Badge, addr: usize, len: usize) -> Result<(), Error>;
}
```

## FaultService (`interface/process.rs`)
用于处理进程故障的回调接口。

```rust
pub trait FaultService {
    /// 处理缺页异常。
    fn page_fault(
        &mut self,
        badge: Badge,
        addr: usize,
        pc: usize,
        cause: usize,
    ) -> Result<(), Error>;

    /// 处理未知故障。
    fn unknown_fault(
        &mut self,
        badge: Badge,
        cause: usize,
        value: usize,
        pc: usize,
    ) -> Result<(), Error>;

    /// 处理非法指令。
    fn illegal_instrution(&mut self, badge: Badge, inst: usize, pc: usize) -> Result<(), Error>;

    /// 处理断点或调试异常。
    fn breakpoint(&mut self, badge: Badge, pc: usize) -> Result<(), Error>;

    /// 处理内存访问故障。
    fn access_fault(&mut self, badge: Badge, addr: usize, pc: usize) -> Result<(), Error>;

    /// 处理未对齐内存访问。
    fn access_misaligned(&mut self, badge: Badge, addr: usize, pc: usize) -> Result<(), Error>;

    /// 处理由内核捕获的系统调用。
    fn syscall(&mut self, badge: Badge, regs: MsgArgs) -> Result<(), Error>;
}
```
