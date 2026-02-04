# 系统调用接口 (System Call Interface)

## 1. 概述

Glenda 使用统一的、基于 Capability 的调用模型。所有内核服务均通过 `ecall` 指令调用 **Capability 指针 (CPtr)** 来访问。

### 寄存器约定 (RISC-V)

| 寄存器 | 名称 | 描述 |
| :--- | :--- | :--- |
| **`a0`** | `cptr` | 被调用对象的 Capability 指针。 |
| **`a7`** | `method` | 方法 ID (取决于对象类型)。 |
| **`a1 - a6`** | `args` | 参数 (存储在 UTCB MR0-MR5 中，但部分快速路径可直接使用寄存器)。 |

> 注意：参数通常由用户库 (`libglenda`) 编组到 UTCB 中，但为了性能，某些调用可能直接使用寄存器传递参数。

## 2. 对象方法

### 2.1 IPC 端点 (`Endpoint`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`Send`** | 1 | - | 发送 IPC 消息。若无接收者则阻塞。 |
| **`Recv`** | 2 | - | 等待 IPC 消息。 |
| **`Call`** | 3 | - | 发送并等待回复 (RPC)。 |
| **`Notify`**| 4 | - | 发送通知 (Signal)。 |

### 2.2 线程控制块 (`TCB`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`Configure`** | 1 | `cspace, vspace, utcb, tf, kstack` | 设置能力空间和相关缓冲区。 |
| **`SetPriority`**| 2 | `prio` | 设置调度优先级 (0-255)。 |
| **`SetEntry`**   | 3 | `pc, sp, tp` | 设置线程入口点、栈指针和 TLS。 |
| **`SetFaultHandler`**| 4 | `ep_cptr` | 设置异常处理端点。 |
| **`SetAffinity`**| 5 | `cpu_id` | 设置 CPU 亲和性。 |
| **`SetRegisters`**| 6 | `regs...` | 写入通用寄存器。 |
| **`Resume`** | 7 | - | 恢复线程执行。 |
| **`Suspend`** | 8 | - | 挂起线程执行。 |

### 2.3 Capability 节点 (`CNode`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`Mint`** | 1 | `src, dest, badge, rights` | 创建带有可选 Badge/权限的新 Cap。 |
| **`Copy`** | 2 | `src, dest, rights` | 复制 Cap 到另一槽位。 |
| **`Delete`** | 3 | `slot` | 删除槽位中的 Cap。 |
| **`Revoke`** | 4 | `slot` | 递归删除该 Cap 的所有派生副本。 |
| **`Debug`** | 5 | - | 输出 CNode 信息到内核日志。 |

### 2.4 虚拟地址空间 (`VSpace`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`Map`** | 1 | `frame_cap, vaddr, flags` | 将 Frame 映射到虚拟地址空间。 |
| **`Unmap`** | 2 | `vaddr, size` | 解除虚拟内存映射。 |
| **`MapTable`**| 3 | `table_cap, vaddr, level` | 映射下一级页表。 |
| **`Setup`** | 4 | - | 初始化 VSpace 根页表。 |
| **`Debug`** | 5 | - | 输出 PageTable 信息到内核日志。 |

### 2.5 页表 (`PageTable`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`MapTable`**| 1 | `table_cap, vaddr, level` | 链接子页表。 |

### 2.6 非类型化内存 (`Untyped`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`Retype`** | 1 | `type, flags, cnode, slot` | 从非类型化内存创建内核对象。 |

### 2.7 中断处理程序 (`IrqHandler`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`SetNotify`**| 1 | `ep_cptr` | 将 IRQ 绑定到通知端点。 |
| **`Ack`** | 2 | - | 确认中断 (EOI)。 |
| **`Clear`** | 3 | - | 解除通知绑定。 |
| **`SetPriority`**| 4 | `prio` | 设置 IRQ 硬件优先级。 |

### 2.8 内核资源 (`Kernel`)

| 方法 | ID | 参数 | 描述 |
| :--- | :-- | :--- | :--- |
| **`PutStr`** | 1 | - | 输出字符串到调试控制台。 |
| **`GetChar`** | 2 | - | 从调试控制台读取字符。 |
| **`GetStr`** | 3 | - | 从调试控制台读取字符串。 |
| **`Shell`** | 4 | - | 进入内核调试 Shell。 |
