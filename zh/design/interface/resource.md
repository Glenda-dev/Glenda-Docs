# 内核资源接口 (Kernel Resource Interfaces)

## 简介
定义了用于管理基础内核能力的 Trait，如虚拟地址空间 (VSpace)、能力空间 (CSpace) 和非类型化内存 (Resources)。

## VSpaceService (`interface/vspace.rs`)
负责管理虚拟内存映射。

```rust
pub trait VSpaceService {
    /// 将帧映射到虚拟地址。
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

    /// 解除内存映射并释放资源。
    fn unmap(
        &mut self,
        vaddr: usize,
        pages: usize,
        objects: &mut dyn ResourceService,
        cnode: CNode,
    ) -> Result<(), Error>;
    
    /// 映射暂存内存。
    fn map_scratch(
        &mut self,
        frame: Frame,
        perms: Perms,
        pages: usize,
        objects: &mut dyn ResourceService,
        slots: &mut dyn CSpaceService,
        dest_cnode: CNode,
    ) -> Result<usize, Error>;

    /// 检查虚拟地址是否已映射。
    fn is_mapped(&self, vaddr: usize, level: usize) -> bool;
}
```

## CSpaceService (`interface/cspace.rs`)
负责管理能力槽 (Capability Slots)。

```rust
pub trait CSpaceService {
    /// 分配能力槽。
    fn alloc(&mut self, objects: &mut dyn ResourceService) -> Result<CapPtr, Error>;
}
```

## ResourceService (`interface/resource.rs`)
负责从非类型化内存中分配内核对象。

```rust
pub trait ResourceService {
    /// 分配内核对象。
    fn alloc(
        &mut self,
        obj_type: CapType,
        flags: usize,
        dest_cnode: CNode,
        dest_slot: CapPtr,
    ) -> Result<(), Error>;

    /// 释放内核能力。
    fn free(&mut self, cap: CapPtr) -> Result<(), Error>;
}
```
