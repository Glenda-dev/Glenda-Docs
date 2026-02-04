# Kernel Resource Interfaces

## Introduction
Defines traits for managing fundamental kernel capabilities like Virtual Address Spaces (VSpace), Capability Spaces (CSpace), and Untyped Memory (Resources).

## VSpaceService (`interface/vspace.rs`)
Responsible for managing virtual memory mappings.

```rust
pub trait VSpaceService {
    /// Map a frame into a virtual address.
    fn map_frame(
        &mut self,
        frame: Frame,
        vaddr: usize,
        perms: Perms,
        pages: usize,
        objects: &mut dyn ResourceService,
        slots: &mut dyn CSpaceService,
        dest_cnode: CNode,
    ) -> Result<(), Error>;

    /// Unmap memory and free resources.
    fn unmap(
        &mut self,
        vaddr: usize,
        pages: usize,
        objects: &mut dyn ResourceService,
        cnode: CNode,
    ) -> Result<(), Error>;
    
    /// Map scratch memory.
    fn map_scratch(
        &mut self,
        frame: Frame,
        perms: Perms,
        pages: usize,
        objects: &mut dyn ResourceService,
        slots: &mut dyn CSpaceService,
        dest_cnode: CNode,
    ) -> Result<usize, Error>;

    /// Check if a virtual address is mapped.
    fn is_mapped(&self, vaddr: usize, level: usize) -> bool;
}
```

## CSpaceService (`interface/cspace.rs`)
Responsible for managing capability slots.

```rust
pub trait CSpaceService {
    /// Allocate a capability slot.
    fn alloc(&mut self, objects: &mut dyn ResourceService) -> Result<CapPtr, Error>;
}
```

## ResourceService (`interface/resource.rs`)
Responsible for allocating kernel objects from untyped memory.

```rust
pub trait ResourceService {
    /// Allocate a kernel object.
    fn alloc(
        &mut self,
        obj_type: CapType,
        flags: usize,
        dest_cnode: CNode,
        dest_slot: CapPtr,
    ) -> Result<(), Error>;

    /// Free a kernel capability.
    fn free(&mut self, cap: CapPtr) -> Result<(), Error>;
}
```
