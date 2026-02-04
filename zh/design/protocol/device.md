# 设备协议定义 (Device Protocol)

## Protocol Label Range
`0x300 - 0x3FF`

## 描述
用于设备发现、驱动加载和硬件资源访问的协议。由 **Unicorn** 管理。

## 指令 (Commands)

### 管理 (Management)
*   `SCAN_BUS` (1): 扫描系统总线 (PCI, Platform) 以查找设备。
*   `LOAD_DRIVER` (2): 为设备加载特定驱动程序。
*   `LIST_DEVICES` (3): 列出已发现的设备。
*   `INIT_MANIFEST` (4): 应用设备树或 ACPI 清单。
*   `GET_DEVICE_BY_NAME` (5): 通过名称解析设备句柄。

### 驱动操作 (Driver Operations)
*   `GET_INFO` (10): 获取设备信息 (厂商/设备 ID)。
*   `MAP_MMIO` (11): 请求设备的 MMIO 映射。
*   `GET_IRQ` (12): 请求设备的中断 Capability。
*   `ALLOC_DMA` (13): 分配 DMA 安全的连续内存。

## 常量 (Constants)

### 总线类型 (Bus Types)
*   `BUS_PCI` (1)
*   `BUS_PLATFORM` (2)

## 结构体 (Structures)

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
