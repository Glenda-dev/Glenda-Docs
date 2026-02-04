# Process Interfaces

## Introduction
Defines traits for process lifecycle management, memory control, and exception handling.

## ProcessService (`interface/process.rs`)
High-level process control methods.

```rust
pub trait ProcessService {
    /// Spawn a new process by name.
    fn spawn(&mut self, name: &str) -> Result<usize, Error>;
    
    /// Fork the current process.
    fn fork(&mut self, pid: Badge) -> Result<usize, Error>;
    
    /// Terminate a process.
    fn exit(&mut self, pid: Badge, code: usize) -> Result<(), Error>;
    
    /// Load an ELF binary image into a target process's address space.
    fn load_image(&mut self, pid: Badge, elf_data: &[u8]) -> Result<(usize, usize), Error>;
}
```

## MemoryService (`interface/memory.rs`)
System-level memory operations for processes.

```rust
pub trait MemoryService {
    /// Adjust the data segment size (heap).
    fn brk(&mut self, pid: Badge, incr: isize) -> Result<usize, Error>;
    
    /// Map memory into the address space.
    fn mmap(&mut self, pid: Badge, addr: usize, len: usize) -> Result<usize, Error>;
    
    /// Unmap memory from the address space.
    fn munmap(&mut self, pid: Badge, addr: usize, len: usize) -> Result<(), Error>;
}
```

## FaultService (`interface/process.rs`)
Callback interface for handling faults for processes.

```rust
pub trait FaultService {
    /// Handle a page fault.
    fn page_fault(
        &mut self,
        badge: Badge,
        addr: usize,
        pc: usize,
        cause: usize,
    ) -> Result<(), Error>;

    /// Handle an unknown fault.
    fn unknown_fault(
        &mut self,
        badge: Badge,
        cause: usize,
        value: usize,
        pc: usize,
    ) -> Result<(), Error>;

    /// Handle an illegal instruction.
    fn illegal_instrution(&mut self, badge: Badge, inst: usize, pc: usize) -> Result<(), Error>;

    /// Handle a breakpoint or debug exception.
    fn breakpoint(&mut self, badge: Badge, pc: usize) -> Result<(), Error>;

    /// Handle a memory access fault.
    fn access_fault(&mut self, badge: Badge, addr: usize, pc: usize) -> Result<(), Error>;

    /// Handle a misaligned memory access.
    fn access_misaligned(&mut self, badge: Badge, addr: usize, pc: usize) -> Result<(), Error>;

    /// Handle a system call trapped by the kernel.
    fn syscall(&mut self, badge: Badge, regs: MsgArgs) -> Result<(), Error>;
}
```
