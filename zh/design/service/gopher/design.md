# Gopher 设计文档

## 1. 简介
Gopher 是 Glenda 的 **网络栈提供者 (Network Stack Provider)**。它实现 TCP/IP 协议栈（基于 LWIP 或类似）并管理网络接口。它通过 **9P2000** 协议（服务 `/net` 层次结构）暴露网络栈，并通过 **专用 IPC (Dedicated IPC)** 提供高性能数据包 I/O。

## 2. 职责

*   **TCP/IP 协议栈**:
    *   实现核心协议：Ethernet, ARP, IP, ICMP, UDP, TCP。
    *   管理路由表和网络配置。
*   **9P2000 服务器 (`/net`)**:
    *   暴露 Plan 9 风格的网络接口 (`/net`)。
    *   提供 `/net/tcp`, `/net/udp`, `/net/ipifc` 等文件。
    *   处理 `open`, `read`, `write`, `control` 消息以进行套接字配置。
*   **网络接口管理**:
    *   与 **Unicorn** 交互以驱动网络硬件 (NIC)。
    *   管理 IP 地址 (DHCP/静态)。
*   **专用 IPC 支持**:
    *   提供共享内存环形缓冲区，用于与 **APE** (应用程序) 和 **Unicorn** (驱动程序) 进行零拷贝数据包传输。

## 3. 架构

Gopher 作为用户空间服务运行。

### 3.1 数据平面 (专用 IPC)
对于高带宽数据应用，9P 传输开销过高。
*   **数据包环**: Gopher 与客户端（通过 APE）建立共享内存环形缓冲区用于 TX/RX。
*   **事件通知**: 使用轻量级 IPC 信号 (Notifications) 唤醒读者/写者。

### 3.2 控制平面 (9P2000)
所有配置和连接建立都通过 9P 进行。
*   **Connect**: 打开 `/net/tcp/clone` 获取新的连接目录。
*   **Configure**: 向 `ctl` 文件写入 "connect IP PORT"。
*   **Status**: 读取 `status` 文件。

### 3.3 硬件交互
Gopher 通过专用 IPC 从 **Unicorn** 管理的 NIC 驱动程序接收（或发送）原始以太网帧。

## 4. 协议接口 (9P2000)

Gopher 服务于 `/net` 命名空间。

### 示例
*   `/net/tcp/clone` - 打开以创建新的 TCP 连接。
*   `/net/tcp/0/ctl`
*   `/net/tcp/0/data`
*   `/net/udp/...`
*   `/net/dns`

## 5. 后端驱动
Gopher 不直接驱动硬件。它与 **Unicorn** 管理的设备驱动程序通信。
