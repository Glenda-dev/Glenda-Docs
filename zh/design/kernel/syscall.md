# 系统调用接口设计

## 1. 概述

Glenda 微内核使用统一的、基于 capability 的调用模型。所有内核服务不是通过系统调用号（例如在 `a7` 中）索引的传统系统调用表访问，而是通过调用 **Capability 指针 (CPtr)** 访问。

## 2. 统一调用接口

通过 `ecall` 指令进入内核的入口点只有一个。内核根据寄存器中提供的 capability 确定操作。

### 寄存器约定 (RISC-V)

| 寄存器 | 名称 | 描述 |
| :--- | :--- | :--- |
| **`a0`** | `cptr` | 指向目标内核对象的 Capability 指针。 |
| **`a7`** | `method` | 方法 ID |
| **`a1 - a6`** | `args / MRs` | 用于参数或 IPC 数据消息寄存器 (MR0-MR5)。 |

## 3. 内核对象方法

当调用 capability 时，`Method ID` 决定操作。

### 3.1 IPC Endpoint
| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Send`** | 1 | 向端点发送 IPC 消息。 |
| **`Recv`** | 2 | 从端点接收 IPC 消息。 |
| **`Call`** | 3 | 发送消息并等待回复 (RPC)。 |
| **`Notify`** | 4 | 向端点发送通知（信号）。 |

### 3.2 Reply 对象
| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Reply`** | 1 | 回复 `Call` 操作。 |

### 3.3 线程控制块 (TCB)

| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Configure`** | 1 | 设置 CSpace, VSpace, UTCB 和 Fault Handler。 |
| **`SetPriority`** | 2 | 设置线程的调度优先级。 |
| **`SetRegisters`** | 3 | 设置线程的 CPU 寄存器。 |
| **`Resume/Suspend`** | 4/5 | 控制线程执行状态。 |

### 3.3 Page Table

| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Map`** | 1 | 将物理页映射到页表。 |
| **`Unmap`** | 2 | 从页表取消映射页。 |

### 3.4 CNode (Capability 空间)

| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Mint`** | 1 | 创建具有指定权限的新 capability。 |
| **`Copy`** | 2 | 复制现有的 capability。 |
| **`Delete`** | 3 | 删除 capability。 |
| **`Revoke`** | 4 | 撤销 capability 及其派生。 |

### 3.5 Untyped 内存
| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`Retype`** | 1 | 将 untyped 内存重类型化为内核对象。 |

### 3.6 IRQ Handler
| 方法 | ID | 描述 |
| :--- | :--- | :--- |
| **`SetNotification`** | 1 | 设置 IRQ 的通知端点。 |
| **`Acknowledge`** | 2 | 确认 IRQ。 |
| **`ClearNotification`** | 3 | 清除 IRQ 的通知端点。 |
| **`SetPriority`** | 4 | 设置 IRQ 优先级。 |
