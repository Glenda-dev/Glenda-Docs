# libglenda-rs: Glenda User-space Development Library

## Progress

- [x] Basic system call wrappers (`sys_invoke`, `sys_send`, `sys_recv`)
- [x] Capability Pointer (`CapPtr`) abstraction
- [x] Kernel object method wrappers (TCB, CNode, PageTable, Untyped, IPC)
- [x] UTCB (User Thread Control Block) structure and access
- [x] Error codes and rights constants
- [ ] C compatible interface (`extern "C"`)
## Usage Example

```rust
use libglenda::caps::CapPtr;
use libglenda::types::*;

fn main() {
    let utcb = libglenda::get_utcb();
    let cspace = CapPtr::new(CSPACE_SLOT);
    
    // Retype some memory
    let untyped = CapPtr::new(MEM_SLOT);
    untyped.untyped_retype(CapType::TCB, 0, 1, cspace, 10);
}
```
