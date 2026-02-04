# Gopher Design Document

## 1. Introduction
Gopher is the **Network Stack Provider** for Glenda. It implements the TCP/IP protocol stack (based on LWIP or similar) and manages network interfaces. It exposes the network stack via the **9P2000** protocol (serving the `/net` hierarchy) and provides high-performance packet I/O via **Dedicated IPC**.

## 2. Responsibilities

*   **TCP/IP Stack**:
    *   Implements core protocols: Ethernet, ARP, IP, ICMP, UDP, TCP.
    *   Manages routing tables and network configuration.
*   **9P2000 Server (`/net`)**:
    *   Exposes the Plan 9-style network interface (`/net`).
    *   Files like `/net/tcp`, `/net/udp`, `/net/ipifc`.
    *   Handles `open`, `read`, `write`, `control` messages for socket configuration.
*   **Network Interface Management**:
    *   Interacts with **Unicorn** to drive network hardware (NICs).
    *   Manages IP addresses (DHCP/Static).
*   **Dedicated IPC Support**:
    *   Provides shared memory rings for zero-copy packet transmission with **APE** (applications) and **Unicorn** (drivers).

## 3. Architecture

Gopher runs as a user-space service.

### 3.1 Data Plane (Dedicated IPC)
For high-bandwidth data applications, the 9P transport overhead is too high.
*   **Packet Rings**: Gopher establishes shared memory ring buffers with clients (via APE) for TX/RX.
*   **Event Notification**: Uses lightweight IPC signals (Notifications) to wake up reader/writer.

### 3.2 Control Plane (9P2000)
All configuration and connection setup happens via 9P.
*   **Connect**: Open `/net/tcp/clone` to get a new connection directory.
*   **Configure**: Write "connect IP PORT" to the `ctl` file.
*   **Status**: Read `status` file.

### 3.3 Hardware Interaction
Gopher receives raw Ethernet frames from **Unicorn**-managed NIC drivers via dedicated IPC.

## 4. Protocol Interface (9P2000)

Gopher serves the `/net` namespace.

### Examples
*   `/net/tcp/clone` - Open to create new TCP connection.
*   `/net/tcp/0/ctl`
*   `/net/tcp/0/data`
*   `/net/udp/...`
*   `/net/dns`

## 5. Backend Drivers
Gopher does not drive hardware directly. It communicates with device drivers managed by **Unicorn**.
