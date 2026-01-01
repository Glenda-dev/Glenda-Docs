
# Interaction Flows

## File Open (`open`)
1. **App** calls `open("/etc/motd")` in libc.
2. **libc** sends a `PosixRequest` to Tux via IPC.
3. **Tux** receives the request and identifies the caller via its **Badge**.
4. **Tux** forwards the path lookup to **Gopher (VFS)**.
5. **Gopher** returns a new `Endpoint Capability` for the file session.
6. **Tux** allocates a new FD (e.g., `3`), stores the capability in the process's `fd_table`, and returns `3` to the App.

## Process Creation (`fork`)
1. **App** calls `fork()`.
3. **Tux** receives the request and identifies the caller via its **Badge**.
4. **Tux** forwards the path lookup to **Factotum (Process Control)**.
5. **Factotum** clones the address space (COW), creates a new process entry, and returns the new PID to Tux.
6. **Tux** duplicates the FD table for the new process.
7. **Tux** creates a new thread in the new process via Factotum.
8. Both parent and child processes resume execution from the `fork()` call, with the child receiving `0` and the parent receiving the child's PID.
