# libglenda-rs: Glenda 微内核开发库

`libglenda-rs` 是 Glenda 微内核的核心开发库。它提供了高级 Rust 抽象和稳定的 C 兼容接口，用于与内核的基于 capability 的系统进行交互。

## 1. 设计目标

- **类型安全**: 利用 Rust 的类型系统在编译时防止无效的 capability 使用。
- **零开销**: 对原始系统调用的最小抽象。
- **C 兼容性**: 提供标准 C API (`extern "C"`) 以支持用 C/C++ 编写的驱动程序和应用程序。
- **RAII 支持**: capability 槽和内存映射的自动资源管理。

## 2. 核心原语

### 2.1 Capability 指针 (CapPtr)
`CapPtr` 是 `usize` 的透明包装器，表示线程 CSpace 中的索引。

```rust
#[repr(transparent)]
pub struct CapPtr(pub usize);
```

### 2.2 消息标签 (MsgTag)
用于描述 IPC 消息的结构。它编码标签、消息长度和标志（如 `HAS_CAP`）。

```rust
pub struct MsgTag(pub usize);
```

### 2.3 用户线程控制块 (UTCB)
UTCB 映射在固定的虚拟地址 (`0x8000_0000`)。它包含消息寄存器和传输描述符。

```rust
#[repr(C)]
pub struct UTCB {
    pub msg_tag: MsgTag,
    pub mrs_regs: [usize; 7],
    pub cap_transfer: usize,
    pub recv_window: usize,
    pub tls: usize,
    pub ipc_buffer_size: usize,
}
```

## 3. Rust API

### 3.1 系统调用
该库提供了 RISC-V `ecall` 的低级包装器：
- `sys_invoke(cptr, method, arg0..arg5)`

### 3.2 面向对象的调用
`CapPtr` 为常见的内核对象操作提供了方法：

- **TCB**: `tcb_configure(cspace, vspace, utcb_vaddr, fault_ep, utcb_frame)`, `tcb_set_priority`, `tcb_resume`, `tcb_suspend`.
- **CNode**: `cnode_mint`, `cnode_copy`, `cnode_delete`, `cnode_revoke`.
- **Untyped**: `untyped_retype`.
- **PageTable**: `pagetable_map`, `pagetable_unmap`.
- **IPC**: `send`, `recv`.

## 4. C 语言接口

`libglenda-rs` 使用 `#[no_mangle]` 导出一组 C 兼容函数。这些旨在供 C 应用程序通过生成的 `glenda.h` 头文件使用。

### 3.1 用户线程控制块 (UTCB)

UTCB 是内核和用户线程之间共享的一页内存。它用于传递大消息和存储线程本地状态。

```c
typedef struct {
    uintptr_t msg_tag;      // 消息标签
    uintptr_t mrs_regs[7];  // 消息寄存器
    uintptr_t cap_transfer; // capability 传输槽
    uintptr_t recv_window;  // 接收窗口描述符
    uintptr_t tls;          // TLS 指针
    uintptr_t ipc_buffer_size;
} glenda_utcb_t;

// 访问 UTCB
glenda_utcb_t* glenda_get_utcb(void);
```

### 3.2 基本类型 (C)

```c
typedef uintptr_t glenda_cptr_t;

typedef struct {
    uintptr_t label;
    uint8_t msg_len;
    bool has_cap;
} glenda_msg_tag_t;
```

### 3.2 系统调用包装器

#### `glenda_syscall_invoke`
在 capability 上调用方法。

```c
uintptr_t glenda_syscall_invoke(
    uintptr_t cptr,
    uintptr_t method,
    uintptr_t arg0,
    uintptr_t arg1,
    uintptr_t arg2,
    uintptr_t arg3,
    uintptr_t arg4,
    uintptr_t arg5
);
```

#### `glenda_ipc_send`
向端点发送消息。

```rust
#[no_mangle]
pub extern "C" fn glenda_ipc_send(
    dest: usize,
    tag: glenda_msg_tag_t,
) -> isize;
```

### 3.3 对象特定的辅助函数

#### Endpoint 操作
```c
// C API
int glenda_endpoint_send(glenda_cptr_t ep, const void* data, size_t len);
int glenda_endpoint_recv(glenda_cptr_t ep, void* data, size_t* len);
```

#### CNode 操作
```c
// C API
int glenda_cnode_copy(glenda_cptr_t cnode, glenda_cptr_t src, glenda_cptr_t dest);
int glenda_cnode_mint(glenda_cptr_t cnode, glenda_cptr_t src, glenda_cptr_t dest, uintptr_t badge);
```

## 4. 内存管理

该库提供了 `VSpace` 抽象来管理进程的地址空间。

- **Rust**: 具有 `map_frame` 等方法的 `VSpace` 结构。
- **C**: `glenda_vspace_map(glenda_cptr_t frame, uintptr_t vaddr, uint32_t rights)`.

## 5. 运行时与启动

`libglenda-rs` 包含 `crt0` (C Runtime 0) 逻辑和核心 Rust 运行时支持：
1.  **目标**: 编译为 `riscv64gc-unknown-none-elf` 以直接在微内核上运行。
2.  **Panic Handler**: 提供默认的 `#[panic_handler]`，进入无限循环（或最终发送故障 IPC）。
3.  **入口点**: 来自内核的入口点。
4.  **初始化**: 初始化 `UTCB` (用户线程控制块) 指针。
5.  **栈与堆**: 设置栈和堆。
6.  **Main**: 调用 `main`。

## 6. 错误处理

所有函数返回有符号整数，其中负值表示 `glenda/errors.h` 中定义的错误代码。

| 错误代码 | 描述 |
| :--- | :--- |
| `0` | 成功 |
| `1` | 无效 Capability |
| `2` | 权限被拒绝 |
| `3` | 无效 Endpoint |
| `4` | 无效对象类型 |
| `5` | 无效方法 |
| `8` | 内存不足 (Untyped OOM) |

## 7. 头文件生成

要生成 C 头文件，请在 `libglenda-rs` 目录中使用 `cbindgen`：

```bash
cbindgen --config cbindgen.toml --crate libglenda-rs --output include/glenda.h
```

## 8. C 使用示例

```c
#include <glenda.h>
#include <stdio.h>

int main() {
    glenda_cptr_t ep = 10; // 假设 10 是 Endpoint cap
    const char* msg = "Hello from C!";
    
    // 在 UTCB 中设置消息
    glenda_utcb_t* utcb = glenda_get_utcb();
    memcpy(utcb->mr, msg, strlen(msg));
    
    glenda_msg_tag_t tag = { .label = 0, .msg_len = strlen(msg) };
    intptr_t result = glenda_ipc_send(ep, tag);
    
    if (result < 0) {
        printf("IPC failed: %ld\n", result);
    }
    
    return 0;
}
```
