# 服务接口 (Server Interfaces)

## 简介
用于实现系统服务和处理 IPC 的接口。

## SystemService (`interface/system.rs`)
系统服务生命周期和分发的核心接口。

```rust
pub trait SystemService {
    /// 初始化服务。
    fn init(&mut self) -> Result<(), Error>;
    
    /// 在端点上监听传入消息。
    fn listen(&mut self, ep: Endpoint, reply: CapPtr) -> Result<(), Error>;
    
    /// 运行主服务循环。
    fn run(&mut self) -> Result<(), Error>;
    
    /// 根据标签和协议分发传入的 IPC 消息。
    fn dispatch(
        &mut self,
        badge: Badge,
        label: usize,
        proto: usize,
        flags: MsgFlags,
        msg: MsgArgs,
    ) -> Result<MsgArgs, Error>;
    
    /// 回复 IPC 消息。
    fn reply(
        &mut self,
        label: usize,
        proto: usize,
        flags: MsgFlags,
        msg: MsgArgs,
    ) -> Result<(), Error>;
    
    /// 退出服务。
    fn exit();
}
```
