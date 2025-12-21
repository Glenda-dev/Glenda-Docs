# libglenda-rs: Glenda Microkernel Development Library

`libglenda-rs` is the core development library for the Glenda microkernel. It provides high-level Rust abstractions and a stable C-compatible interface for interacting with the kernel's capability-based system.

## 1. Design Goals

- **Type Safety**: Leverage Rust's type system to prevent invalid capability usage at compile time.
- **Zero Overhead**: Minimal abstraction over raw system calls.
- **C Compatibility**: Provide a standard C API (`extern "C"`) to support drivers and applications written in C/C++.
- **RAII Support**: Automatic resource management for capability slots and memory mappings.

## 2. Core Primitives

### 2.1 Capability Pointer (CPtr)
A `CPtr` is an integer index into the thread's CSpace.

```rust
#[repr(transparent)]
pub struct CPtr(pub usize);
```

### 2.2 Message Tag (MsgTag)
Used to describe the structure of an IPC message.

```rust
#[repr(C)]
pub struct MsgTag {
    pub label: usize,
    pub msg_len: u8,
    pub cap_transfer: bool,
    // ... other flags
}
```

## 3. C Language Interface

`libglenda-rs` exports a set of C-compatible functions using `#[no_mangle]`. These are intended to be used by C applications via a generated `glenda.h` header.

### 3.1 User Thread Control Block (UTCB)

The UTCB is a page of memory shared between the kernel and the user thread. It is used for passing large messages and storing thread-local state.

```c
typedef struct {
    uintptr_t mr[64];       // Message Registers
    uintptr_t cap_transfer; // Slot for capability transfer
    uintptr_t badge;        // Received badge
    // ...
} glenda_utcb_t;

// Accessing the UTCB
extern glenda_utcb_t* glenda_get_utcb(void);
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

```rust
#[no_mangle]
pub extern "C" fn glenda_syscall_invoke(
    cptr: usize,
    method: usize,
    arg0: usize,
    arg1: usize,
    arg2: usize,
) -> isize;
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

`libglenda-rs` includes the `crt0` (C Runtime 0) logic:
1.  Entry point from kernel.
2.  Initialization of the `UTCB` (User Thread Control Block) pointer.
3.  Setup of the stack and heap.
4.  Calling `main`.

## 6. Error Handling

All functions return a signed integer where negative values represent error codes defined in `glenda/errors.h`.

| Error Code | Description |
| :--- | :--- |
| `0` | Success |
| `-1` | Invalid Capability |
| `-2` | Permission Denied |
| `-3` | Out of Memory |

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
