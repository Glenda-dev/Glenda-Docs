# Device Protocol Definition

## Protocol Label Range
`0x300 - 0x3FF`

## Description
Protocol for device discovery, driver loading, and hardware resource access. Managed by **Unicorn**.

## Commands

### Management
*   `SCAN_BUS` (1): Scan system buses (PCI, Platform) for devices.
*   `LOAD_DRIVER` (2): Load a specific driver for a device.
*   `LIST_DEVICES` (3): List discovered devices.
*   `INIT_MANIFEST` (4): Appply device tree or ACPI manifest.
*   `GET_DEVICE_BY_NAME` (5): Resolve device handle by name.

### Driver Operations
*   `GET_INFO` (10): Get device information (Vendor/Device ID).
*   `MAP_MMIO` (11): Request MMIO mapping for a device BAR.
*   `GET_IRQ` (12): Request IRQ capability for a device.
*   `ALLOC_DMA` (13): Allocate DMA-safe contiguous memory.

## Constants

### Bus Types
*   `BUS_PCI` (1)
*   `BUS_PLATFORM` (2)

## Structures

### DeviceInfo
```rust
#[repr(C)]
pub struct DeviceInfo {
    pub vendor_id: u16,
    pub device_id: u16,
    pub class_code: u8,
    pub subclass: u8,
    pub prog_if: u8,
    pub revision: u8,
    pub bus: u8,
    pub dev: u8,
    pub func: u8,
    pub irq_line: u8,
}
```
