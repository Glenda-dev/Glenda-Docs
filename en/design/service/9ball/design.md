# 9Ball Design Document

## 1. Introduction
9Ball is the Root Task (Process ID 1 equivalent) of the Glenda operating system. It is the first user-space program launched by the kernel. Its primary role is to bootstrap the user-space environment, specifically initializing the `Factotum` service manager, and then coordinating the startup of other essential system services.

## 2. Responsibilities

*   **System Bootstrap**: Initialize the global capability space and essential resources handed over by the kernel.
*   **Factotum Initialization**: Load and start the `Factotum` service. Since `Factotum` is the Task Manager, 9Ball must perform the initial loading of Factotum manually (using `libglenda` helpers) before it can delegate process creation to Factotum.
*   **Service Orchestration**: Once Factotum is running, 9Ball acts as the system configuration manager, instructing Factotum to launch other core services (e.g., Unicorn, Gopher, Rio) based on a configuration.
*   **System Monitoring**: After bootstrap, 9Ball enters a supervision loop, monitoring the liveness of critical services and handling system-wide power states (shutdown/reboot).
*   **Log Management**: Capture and buffer logs from early-boot services before a dedicated logging daemon is available.

## 3. Boot Sequence

### Phase 1: Kernel Handover
The kernel starts 9Ball and grants it all available system resources (Untyped Memory, IRQ capabilities, Device Frames, etc.) via the initial capability space (CSpace).

### Phase 2: Launching Factotum
1.  9Ball locates the `Factotum` binary (typically provided as a boot module by the bootloader).
2.  9Ball creates a new Protection Domain (CSpace and VSpace) for Factotum.
3.  **Resource Delegation**: 9Ball transfers the majority of system resources (Untyped Memory, IRQ control) to Factotum. This allows Factotum to fulfill its role as the memory and task manager for the rest of the system.
4.  9Ball starts the Factotum thread.

### Phase 3: Launching System Services
Once Factotum is active, 9Ball establishes an IPC channel with it. 9Ball then proceeds to launch other services by sending requests to Factotum.

**Workflow for starting a service (e.g., Gopher):**
1.  9Ball reads the service binary in initrd.
2.  9Ball sends a `CreateProcess` IPC message to Factotum, providing the binary data or location.
3.  Factotum parses the ELF, allocates memory, and creates the thread.
4.  Factotum returns the process handle to 9Ball.
5.  9Ball sends a `Start` message to the new process.

### Phase 4: Supervision Loop
After the boot sequence is complete, 9Ball enters a blocking loop waiting for IPC messages or fault notifications.
*   **Fault Handling**: If a critical service crashes, Factotum (as the exception handler) might notify 9Ball.
*   **System Power**: 9Ball handles requests for shutdown or reboot.

## 4. Interfaces

### Interaction with Factotum
9Ball acts as a client to Factotum for process management.

*   **Interface**: `org.glenda.proc.Manager` (Conceptual)
*   **Methods**:
    *   `Spawn(name: str, binary: Blob Address) -> Handle`
    *   `Kill(handle: Handle)`
    *   `GetStatus(handle: Handle) -> Status`

## 5. Critical Services List
9Ball is responsible for ensuring the following services are running:
1.  **Factotum**: Exception & Task Manager (The "Kernel" of user space).
2.  **Unicorn**: Device Driver Manager.
3.  **Gopher**: VFS and Namespace Server.
4.  **Rio**: Windowing and Display Server (optional for headless).

## 6. Log Management

9Ball plays a crucial role in system observability during the boot phase.

### Early Boot Logging
Before the display server (Rio) or the POSIX server (Tux) are active, 9Ball acts as the primary log sink.
1.  **Kernel Console**: 9Ball utilizes the kernel's debug printing facility (e.g., `sys_debug_print`) to output critical boot milestones to the serial console.
2.  **Service Output Capture**: When 9Ball instructs Factotum to spawn a service, it provides a capability for an IPC endpoint or a shared memory ring buffer to be used as the service's `stdout`/`stderr`.

### Log Buffering
9Ball maintains an in-memory **Ring Buffer** (e.g., 64KB) to store logs from all supervised services.
*   This ensures that if a service crashes during boot, its last output is preserved.
*   This buffer can be queried by a developer tool or dumped to the screen if the boot fails.

### Log Handover
Once the system is fully booted, 9Ball can optionally stop acting as the log sink and redirect new service outputs to a dedicated system logger (e.g., `syslogd` running on Tux), or continue to act as the kernel log buffer.

## 7. Future Considerations
*   **Configurable Boot**: Support reading a `init.toml` configuration file to define which services to start and their arguments.
*   **Parallel Startup**: Launching independent services in parallel to speed up boot.
*   **Recovery Policies**: Defining restart policies (always, on-failure, never) for each service.
