# Fossil 设计文档

## 1. 简介
Fossil 是 Glenda 的主 **文件系统服务器 (File System Server)**。它管理块设备上的持久存储，并提供层次化的文件结构。

## 2. 架构

### 2.1 后端抽象
Fossil 通过模块化的后端 Trait 支持多种文件系统实现。
*   **磁盘驱动**: 连接到 **Unicorn** (块设备管理器) 以读写扇区。
*   **支持格式**:
    *   **FAT32**: 用于 EFI 系统分区和互操作性。
    *   **Ext4** (初始只读): Linux 兼容性。
    *   **GlendaFS** (计划中): 针对 Capability 存储优化的日志结构文件系统。

### 2.2 元数据与数据
*   **控制平面 (FS 协议)**: 目录遍历 (`walk`)、文件创建 (`create`) 和状态检查 (`stat`) 通过标准 FS 协议消息处理。
*   **数据平面 (共享内存)**:
    *   大块读/写使用 **类 DMA 的共享内存机制**。
    *   Fossil 在文件打开时暴露一个 "Dataport" Capability。
    *   数据直接从文件缓存复制到客户端缓冲区，无需中间编组。

### 2.3 缓存
Fossil 与 **Warren** 的内存管理器集成。
*   **页缓存**: 使用统一的页缓存，clean 页面可以在内存压力下被内核/Warren 驱逐。

## 3. 接口

### 3.1 文件接口
Fossil 通常服务于根 `/` 或挂载点如 `/mnt/disk`。

### 3.2 块接口 (Unicorn 客户端)
Fossil 消费由 Unicorn 提供的块设备。
*   协议: `READ_BLOCKS`, `WRITE_BLOCKS` (异步 IPC)。

## 4. 与 Gopher 的对比
此前，Gopher 负责 VFS 职责。现在：
*   **Fossil**: 处理 **磁盘** 文件系统。
*   **Gopher**: 处理 **网络** 栈 (`/net`)。
