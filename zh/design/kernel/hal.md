# Glenda 内核引导 (Boot) 设计

## 1. 引导流程概述

Glenda 支持多种引导协议，主要通过 `kernel/src/boot/` 下的模块实现。

1.  **Firmware/Loader**:
    *   **OpenSBI**: 提供 RISC-V 运行环境。
    *   **Limine / UEFI / U-Boot / Multiboot2**: 加载内核镜像。
2.  **Bootstrap (`kernel/src/boot/mod.rs`)**:
    *   由 `linker.ld` 指定的入口点开始。
    *   建立临时内核页表，映射 HHDM。
    *   收集 `BootLoaderInfo` 并存储在 `BOOT_LOADER_INFO` 常量中。
3.  **初始化 (`kernel/src/init/mod.rs`)**:
    *   初始化中断控制器 (PLIC/CLINT)。
    *   初始化物理内存管理器。
    *   启动 Root Task (`Warren`)。

## 2. BootInfo 传递

内核在启动 Root Task 时，会将 `BootLoaderInfo` 中的关键数据序列化为 `BootInfo` 结构体，并映射到用户空间的 `BOOTINFO_VA` 地址。

*   **包含内容**：
    *   物理内存布局 (`MemoryMap`)。
    *   `initrd` 物理地址与大小。
    *   设备树 (DTB) 地址。
    *   帧缓冲区信息 (`FrameBuffer`)。
*   **用户侧解析**：在 `service/warren/src/main.rs` 中直接将 `BOOTINFO_VA` 强转为 `&BootInfo` 进行解析。

## 3. 硬件抽象层 (HAL)

`kernel/src/hal/` 封装了架构相关的寄存器访问和控制：
*   **CPU**: 寄存器保存恢复、Hart 管理。
*   **Timer**: 时钟中断配置（通常通过 SBI）。
*   **Trap**: S-Mode 异常分发处理。
