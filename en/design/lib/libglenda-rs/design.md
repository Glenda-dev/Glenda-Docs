# libglenda-rs: Glenda Microkernel Development Library

`libglenda-rs` is the core development library for the Glenda microkernel. It provides high-level Rust abstractions and a stable C-compatible interface for interacting with the kernel's capability-based system.

## 1. Design Goals

- **Type Safety**: Leverage Rust's type system to prevent invalid capability usage at compile time.
- **Zero Overhead**: Minimal abstraction over raw system calls.
- **C Compatibility**: Provide a standard C API (`extern "C"`) to support drivers and applications written in C/C++.
- **RAII Support**: Automatic resource management for capability slots and memory mappings.

## 2. Core Primitives

### 2.1 Capability Pointer (CapPtr)
A `CapPtr` is a transparent wrapper around a `usize`, representing an index into the thread's CSpace.

```rust
#[repr(transparent)]
pub struct CapPtr(pub usize);
```

### 2.2 Message Tag (MsgTag)
Used to describe the structure of an IPC message. It encodes the label, message length, and flags (like `HAS_CAP`).

```rust
pub struct MsgTag(pub usize);
```

### 2.3 User Thread Control Block (UTCB)
The UTCB is mapped at a fixed virtual address (`0x8000_0000`). It contains message registers and transfer descriptors.

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

### 3.1 System Calls
The library provides low-level wrappers for RISC-V `ecall`:
- `sys_invoke(cptr, method, arg0..arg5)`
- `sys_send(cptr, msg_info)`
- `sys_recv(cptr)`

### 3.2 Object-Oriented Invocation
`CapPtr` provides methods for common kernel object operations:

- **TCB**: `tcb_configure(cspace, vspace, utcb_vaddr, fault_ep, utcb_frame)`, `tcb_set_priority`, `tcb_resume`, `tcb_suspend`.
- **CNode**: `cnode_mint`, `cnode_copy`, `cnode_delete`, `cnode_revoke`.
- **Untyped**: `untyped_retype`.
- **PageTable**: `pagetable_map`, `pagetable_unmap`.
- **IPC**: `send`, `recv`.

## 4. C Language Interface

`libglenda-rs` exports a set of C-compatible functions using `#[no_mangle]`. These are intended to be used by C applications via a generated `glenda.h` header.

### 3.1 User Thread Control Block (UTCB)

The UTCB is a page of memory shared between the kernel and the user thread. It is used for passing large messages and storing thread-local state.

```c
typedef struct {
    uintptr_t msg_tag;      // Message Tag
    uintptr_t mrs_regs[7];  // Message Registers
    uintptr_t cap_transfer; // Slot for capability transfer
    uintptr_t recv_window;  // Receive window descriptor
    uintptr_t tls;          // TLS pointer
    uintptr_t ipc_buffer_size;
} glenda_utcb_t;

// Accessing the UTCB
glenda_utcb_t* glenda_get_utcb(void);
```

### 3.2 Basic Types (C)

```c
typedef uintptr_t glenda_cptr_t;

typedef struct {
    uintptr_t label;
    uint8_t msg_len;
    bool has_cap;
} glenda_msg_tag_t;
```

### 3.2 System Call Wrappers

#### `glenda_syscall_invoke`
Invokes a method on a capability.

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
Sends a message to an endpoint.

```rust
#[no_mangle]
pub extern "C" fn glenda_ipc_send(
    dest: usize,
    tag: glenda_msg_tag_t,
) -> isize;
```

### 3.3 Object-Specific Helpers

#### Endpoint Operations
```c
// C API
int glenda_endpoint_send(glenda_cptr_t ep, const void* data, size_t len);
int glenda_endpoint_recv(glenda_cptr_t ep, void* data, size_t* len);
```

#### CNode Operations
```c
// C API
int glenda_cnode_copy(glenda_cptr_t cnode, glenda_cptr_t src, glenda_cptr_t dest);
int glenda_cnode_mint(glenda_cptr_t cnode, glenda_cptr_t src, glenda_cptr_t dest, uintptr_t badge);
```

## 4. Memory Management

The library provides a `VSpace` abstraction for managing the process's address space.

- **Rust**: `VSpace` struct with methods like `map_frame`.
- **C**: `glenda_vspace_map(glenda_cptr_t frame, uintptr_t vaddr, uint32_t rights)`.

## 5. Runtime & Startup

`libglenda-rs` includes the `crt0` (C Runtime 0) logic and core Rust runtime support:
1.  **Target**: Compiled for `riscv64gc-unknown-none-elf` to run directly on the microkernel.
2.  **Panic Handler**: Provides a default `#[panic_handler]` that enters an infinite loop (or eventually sends a fault IPC).
3.  **Entry Point**: Entry point from kernel.
4.  **Initialization**: Initialization of the `UTCB` (User Thread Control Block) pointer.
5.  **Stack & Heap**: Setup of the stack and heap.
6.  **Main**: Calling `main`.

## 6. Error Handling

All functions return a signed integer where negative values represent error codes defined in `glenda/errors.h`.

| Error Code | Description |
| :--- | :--- |
| `0` | Success |
| `1` | Invalid Capability |
| `2` | Permission Denied |
| `3` | Invalid Endpoint |
| `4` | Invalid Object Type |
| `5` | Invalid Method |
| `8` | Out of Memory (Untyped OOM) |

## 7. Header Generation

To generate the C header file, use `cbindgen` in the `libglenda-rs` directory:

```bash
cbindgen --config cbindgen.toml --crate libglenda-rs --output include/glenda.h
```

## 8. Example C Usage

```c
#include <glenda.h>
#include <stdio.h>

int main() {
    glenda_cptr_t ep = 10; // Assume 10 is an Endpoint cap
    const char* msg = "Hello from C!";
    
    // Set message in UTCB
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
