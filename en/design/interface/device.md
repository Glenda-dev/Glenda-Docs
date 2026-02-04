# Device Interfaces

## Introduction
Defines traits for hardware discovery, PCI bus access, and DMA memory management.

## DeviceService (`interface/device.rs`)
Hardware discovery and management.

```rust
pub trait DeviceService {
    /// Scan the platform topology (e.g., Device Tree).
    fn scan_platform(&mut self, info: &PlatformInfo);
    
    /// Retrieve a device node by ID.
    fn get_node(&self, id: usize) -> Option<&DeviceNode>;
    
    /// Find a device node compatible with a specific string.
    fn find_compatible(&self, compat: &str) -> Option<&DeviceNode>;
}
```

## PciService (`interface/device.rs`)
PCI bus access and scanning.

```rust
pub trait PciService {
    /// Read from PCI Configuration Space.
    fn read_config(&self, bus: u8, dev: u8, func: u8, offset: usize, size: usize) -> u32;
    
    /// Write to PCI Configuration Space.
    fn write_config(&mut self, bus: u8, dev: u8, func: u8, offset: usize, value: u32, size: usize);
    
    /// Scan the PCI bus for devices and register them with the Device Manager.
    fn scan(&mut self, dev_mgr: &mut dyn DeviceService);
}
```

## DmaService (`interface/device.rs`)
DMA-safe memory allocation.

```rust
pub trait DmaService {
    /// Allocate contiguous physical memory suitable for DMA.
    fn alloc_dma(&mut self, size: usize) -> Result<usize, Error>;
    
    /// Free previously allocated DMA memory.
    fn free_dma(&mut self, paddr: usize, size: usize);
}
```
