# Unicorn Design Document

## 1. Introduction
Unicorn is the Device Driver Manager for Glenda. In a microkernel, drivers run as user-space processes. Unicorn is responsible for managing these driver processes, discovering hardware, and routing hardware resources (MMIO, IRQs) to the appropriate drivers.

## 2. Responsibilities

*   **Hardware Discovery**:
    *   Parses the Device Tree (DTB) or scans buses (PCIe, USB) to enumerate available hardware.
*   **Driver Loading**:
    *   Maintains a registry of available drivers (mapping Device ID -> Driver Binary).
    *   Requests Factotum to spawn driver processes.
*   **Resource Delegation**:
    *   Grants MMIO frame capabilities and IRQ handler capabilities to the specific driver process.
*   **IOMMU Management**:
    *   Configures IOMMU groups to isolate devices and protect system memory from rogue DMA.

## 3. Architecture

Unicorn acts as a registry and supervisor. It does *not* perform the I/O itself; it sets up the connections.

### Driver Model
Drivers in Glenda are standalone processes.
*   **Interfaces**: Drivers implement standard interfaces (e.g., `org.glenda.driver.Block`, `org.glenda.driver.Network`).
*   **IPC**: Clients (like Gopher) talk directly to the driver process via IPC after looking up the driver in Unicorn or the Name Server.

## 4. Supported Subsystems
*   **UART**: Serial consoles.
*   **VirtIO**: Network, Block, GPU, Input (for virtualized environments).
*   **PCIe**: Bus enumeration.

## 5. Interfaces

### 5.1 Management Interface
Used by the system (e.g., Shell, Init) to manage drivers.

*   **Interface**: `org.glenda.unicorn.Manager`
*   **Methods**:
    *   `ScanBus(bus_type: BusType)`: Trigger a bus rescan.
    *   `LoadDriver(driver_path: String)`: Manually load a driver.
    *   `ListDevices() -> Vec<DeviceInfo>`: List all discovered devices.

### 5.2 Driver Interface
Used by driver processes to access hardware resources. When Unicorn spawns a driver, it passes a `Device` capability to the new process.

*   **Interface**: `org.glenda.unicorn.Device`
*   **Methods**:
    *   `GetInfo() -> DeviceInfo`: Retrieve device metadata (compatible strings, registers, interrupts).
    *   `MapMmio(index: u32) -> CapPtr<Frame>`: Request a capability to map a specific MMIO region defined in the device tree/BAR.
    *   `GetIrq(index: u32) -> CapPtr<IrqHandler>`: Request a capability to handle a specific interrupt line.
    *   `AllocDma(size: usize) -> DmaRegion`: Allocate physically contiguous memory for DMA.
        *   Returns: `vaddr`, `paddr`, and a `Frame` capability.

## 6. Driver Development Guide

To write a driver for a specific device in Glenda, the driver must implement the logic to interact with the hardware using the resources provided by Unicorn, and expose a service interface (usually via 9P or a specific IPC protocol) to the rest of the system.

### 6.1 Driver Lifecycle
1.  **Entry**: The driver starts as a normal process.
2.  **Handshake**: It receives the `Device` capability from its startup arguments or environment.
3.  **Probe**: It calls `Device::GetInfo` to verify the hardware version/configuration.
4.  **Setup**:
    *   Call `Device::MapMmio` to map registers into the address space.
    *   Call `Device::GetIrq` to get the IRQ handle.
    *   Initialize the hardware.
5.  **Serve**:
    *   Register a service (e.g., mount point in Gopher).
    *   Enter an event loop handling IPC requests and IRQ notifications.

### 6.2 Example: Simple UART Driver (Rust)

```rust
struct UartDriver {
    mmio: volatile::VolatilePtr<u8>,
    irq: CapPtr,
}

impl UartDriver {
    fn new(device: &unicorn::DeviceClient) -> Self {
        // 1. Get MMIO
        let mmio_cap = device.map_mmio(0).expect("Failed to map MMIO");
        let mmio_ptr = vm::map_device(mmio_cap); // Map cap to VSpace

        // 2. Get IRQ
        let irq_cap = device.get_irq(0).expect("Failed to get IRQ");

        Self {
            mmio: unsafe { volatile::VolatilePtr::new(mmio_ptr) },
            irq: irq_cap,
        }
    }

    fn init(&mut self) {
        // Configure UART registers
        self.mmio.write(IER, 0x01); // Enable RX interrupt
    }

    fn irq_handler(&mut self) {
        // Handle interrupt
        let c = self.mmio.read(RBR);
        println!("Received char: {}", c as char);
        // Acknowledge is handled by kernel/Unicorn usually, or specific EOI write
        self.irq.ack(); 
    }

    fn run(&mut self) {
        loop {
            // Wait for IRQ or IPC
            let badge = ipc::wait(self.irq);
            if badge == IRQ_BADGE {
                self.irq_handler();
            }
        }
    }
}
```

### 6.3 DMA Handling
For devices requiring DMA (e.g., VirtIO Network), the driver must allocate DMA memory via Unicorn to ensure the memory is pinned and physically contiguous (if no IOMMU) or properly mapped in the IOMMU.

```rust
let dma_region = device.alloc_dma(4096).unwrap();
// Write physical address to device register
mmio.write(DESC_TABLE_ADDR, dma_region.paddr);
```
