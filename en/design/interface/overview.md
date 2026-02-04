# Development Interfaces Design

## 1. Introduction
This section describes the high-level Rust APIs and Traits provided by `libglenda-rs/interface` for application and service developers.

## 2. Interface Modules

| Module | Description | Reference |
| :--- | :--- | :--- |
| **Process** | Process Lifecycle, Memory, and Faults | [process.md](process.md) |
| **Device** | Hardware, PCI, DMA Access | [device.md](device.md) |
| **Resource** | Kernel Object Management (VSpace, CSpace) | [resource.md](resource.md) |
| **Server** | System Service Framework | [server.md](server.md) |
| **Async** | Async/Await Support | [async.md](async.md) |

## 3. Core Traits Summary

The interface layer abstracts the underlying IPC mechanisms into idiomatic Rust traits. See detailed files for full API listings.
