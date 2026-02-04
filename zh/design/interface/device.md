# 设备接口 (Device Interfaces)

## 简介
定义了用于硬件发现、PCI 总线访问和 DMA 内存管理的 Trait。

## DeviceService (`interface/device.rs`)
硬件发现和管理。

```rust
pub trait DeviceService {
    /// 扫描平台拓扑 (例如设备树)。
    fn scan_platform(&mut self, info: &PlatformInfo);
    
    /// 通过 ID 获取设备节点。
    fn get_node(&self, id: usize) -> Option<&DeviceNode>;
    
    /// 查找与特定字符串兼容的设备节点。
    fn find_compatible(&self, compat: &str) -> Option<&DeviceNode>;
}
```

## PciService (`interface/device.rs`)
PCI 总线访问和扫描。

```rust
pub trait PciService {
    /// 读取 PCI 配置空间。
    fn read_config(&self, bus: u8, dev: u8, func: u8, offset: usize, size: usize) -> u32;
    
    /// 写入 PCI 配置空间。
    fn write_config(&mut self, bus: u8, dev: u8, func: u8, offset: usize, value: u32, size: usize);
    
    /// 扫描 PCI 总线以查找设备并将其注册到设备管理器。
    fn scan(&mut self, dev_mgr: &mut dyn DeviceService);
}
```

## DmaService (`interface/device.rs`)
DMA 安全内存分配。

```rust
pub trait DmaService {
    /// 分配适合 DMA 的连续物理内存。
    fn alloc_dma(&mut self, size: usize) -> Result<usize, Error>;
    
    /// 释放之前分配的 DMA 内存。
    fn free_dma(&mut self, paddr: usize, size: usize);
}
```
