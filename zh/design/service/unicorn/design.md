# Unicorn 设计文档

## 1. 简介
Unicorn 是 Glenda 的设备驱动管理器。在微内核中，驱动程序作为用户空间进程运行。Unicorn 负责管理这些驱动程序进程，发现硬件，并将硬件资源 (MMIO, IRQ) 路由到相应的驱动程序。

## 2. 职责

*   **硬件发现**:
    *   解析设备树 (DTB) 或扫描总线 (PCIe, USB) 以枚举可用硬件。
*   **驱动加载**:
    *   维护可用驱动程序的注册表（映射 Device ID -> Driver Binary）。
    *   请求 Factotum 启动驱动程序进程。
*   **资源委派**:
    *   将 MMIO 帧 capabilities 和 IRQ 处理程序 capabilities 授予特定的驱动程序进程。
*   **IOMMU 管理**:
    *   配置 IOMMU 组以隔离设备并保护系统内存免受恶意 DMA 的侵害。

## 3. 架构

Unicorn 充当注册表和监督者。它 *不* 执行 I/O 本身；它建立连接。

### 驱动模型
Glenda 中的驱动程序是独立的进程。
*   **接口**: 驱动程序实现标准接口（例如 `org.glenda.driver.Block`, `org.glenda.driver.Network`）。
*   **IPC**: 客户端（如 Gopher）在 Unicorn 或名称服务器中查找驱动程序后，通过 IPC 直接与驱动程序进程对话。

### 驱动注册与匹配机制
Unicorn 采用元数据驱动的机制来管理驱动程序与硬件的绑定。

1.  **驱动清单 (Driver Manifest)**:
    每个驱动程序都附带一个清单文件（例如 `manifest.toml`），描述其支持的硬件。
    ```toml
    [driver]
    name = "ns16550a"
    binary = "/bin/drivers/ns16550a"
    type = "process"

    [match]
    compatible = ["ns16550a", "snps,dw-apb-uart"]

    [pci]
    ids = [{ vendor = 0x1234, device = 0x5678 }]
    ```

2.  **匹配流程**:
    *   **注册**: Unicorn 启动时扫描系统驱动目录，解析所有清单并建立匹配表。
    *   **发现**: Unicorn 扫描 DTB 或枚举 PCI 总线以发现硬件设备。
    *   **绑定**: 对于每个设备，Unicorn 在匹配表中查找对应的驱动。如果找到，则请求 Factotum 启动驱动进程，并将设备 Capability 传递给它。

## 4. 支持的子系统
*   **UART**: 串行控制台。
*   **VirtIO**: 网络、块、GPU、输入（用于虚拟化环境）。
*   **PCIe**: 总线枚举。

## 5. 接口

### 5.1 管理接口
供系统（例如 Shell, Init）用于管理驱动程序。

*   **接口**: `org.glenda.unicorn.Manager`
*   **方法**:
    *   `ScanBus(bus_type: BusType)`: 触发总线重新扫描。
    *   `LoadDriver(driver_path: String)`: 手动加载驱动程序。
    *   `ListDevices() -> Vec<DeviceInfo>`: 列出所有发现的设备。

### 5.2 驱动接口
供驱动程序进程用于访问硬件资源。当 Unicorn 启动驱动程序时，它将 `Device` capability 传递给新进程。

*   **接口**: `org.glenda.unicorn.Device`
*   **方法**:
    *   `GetInfo() -> DeviceInfo`: 检索设备元数据（兼容字符串、寄存器、中断）。
    *   `MapMmio(index: u32) -> CapPtr<Frame>`: 请求 capability 以映射设备树/BAR 中定义的特定 MMIO 区域。
    *   `GetIrq(index: u32) -> CapPtr<IrqHandler>`: 请求 capability 以处理特定的中断线。
    *   `AllocDma(size: usize) -> DmaRegion`: 为 DMA 分配物理连续内存。
        *   返回: `vaddr`, `paddr`, 和 `Frame` capability。

## 6. 驱动开发指南

要在 Glenda 中为特定设备编写驱动程序，驱动程序必须实现使用 Unicorn 提供的资源与硬件交互的逻辑，并向系统的其余部分暴露服务接口（通常通过 9P 或特定的 IPC 协议）。

### 6.1 驱动生命周期
1.  **入口**: 驱动程序作为普通进程启动。
2.  **握手**: 它从启动参数或环境中接收 `Device` capability。
3.  **探测**: 它调用 `Device::GetInfo` 来验证硬件版本/配置。
4.  **设置**:
    *   调用 `Device::MapMmio` 将寄存器映射到地址空间。
    *   调用 `Device::GetIrq` 获取 IRQ 句柄。
    *   初始化硬件。
5.  **服务**:
    *   注册服务（例如，在 Gopher 中挂载点）。
    *   进入处理 IPC 请求和 IRQ 通知的事件循环。

### 6.2 示例: 简单 UART 驱动 (Rust)

```rust
struct UartDriver {
    mmio: volatile::VolatilePtr<u8>,
    irq: CapPtr,
}

impl UartDriver {
    fn new(device: &unicorn::DeviceClient) -> Self {
        // 1. 获取 MMIO
        let mmio_cap = device.map_mmio(0).expect("Failed to map MMIO");
        let mmio_ptr = vm::map_device(mmio_cap); // 将 cap 映射到 VSpace

        // 2. 获取 IRQ
        let irq_cap = device.get_irq(0).expect("Failed to get IRQ");

        Self {
            mmio: unsafe { volatile::VolatilePtr::new(mmio_ptr) },
            irq: irq_cap,
        }
    }

    fn init(&mut self) {
        // 配置 UART 寄存器
        self.mmio.write(IER, 0x01); // 启用 RX 中断
    }

    fn irq_handler(&mut self) {
        // 处理中断
        let c = self.mmio.read(RBR);
        println!("Received char: {}", c as char);
        // 确认通常由内核/Unicorn 处理，或特定的 EOI 写入
        self.irq.ack(); 
    }

    fn run(&mut self) {
        loop {
            // 等待 IRQ 或 IPC
            let badge = ipc::wait(self.irq);
            if badge == IRQ_BADGE {
                self.irq_handler();
            }
        }
    }
}
```

### 6.3 DMA 处理
对于需要 DMA 的设备（例如 VirtIO 网络），驱动程序必须通过 Unicorn 分配 DMA 内存，以确保存储器被固定且物理连续（如果没有 IOMMU）或在 IOMMU 中正确映射。

```rust
let dma_region = device.alloc_dma(4096).unwrap();
// 将物理地址写入设备寄存器
mmio.write(DESC_TABLE_ADDR, dma_region.paddr);
```
