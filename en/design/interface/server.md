# Server Interfaces

## Introduction
Interfaces for implementing system services and handling IPC.

## SystemService (`interface/system.rs`)
The core interface for system service lifecycle and dispatch.

```rust
pub trait SystemService {
    /// Initialize the service.
    fn init(&mut self) -> Result<(), Error>;
    
    /// Listen on an endpoint for incoming messages.
    fn listen(&mut self, ep: Endpoint, reply: CapPtr) -> Result<(), Error>;
    
    /// Run the main service loop.
    fn run(&mut self) -> Result<(), Error>;
    
    /// Dispatch an incoming IPC message based on its label and protocol.
    fn dispatch(
        &mut self,
        badge: Badge,
        label: usize,
        proto: usize,
        flags: MsgFlags,
        msg: MsgArgs,
    ) -> Result<MsgArgs, Error>;
    
    /// Reply to an IPC message.
    fn reply(
        &mut self,
        label: usize,
        proto: usize,
        flags: MsgFlags,
        msg: MsgArgs,
    ) -> Result<(), Error>;
    
    /// Exit the service.
    fn exit();
}
```
