# Capability System (Cap) Design

## 1. Philosophy: Object-Capability Model

Glenda uses a Capability-based access control model.
*   **No Global IDs**: There are no global PIDs or file descriptors accessible by default.
*   **Explicit Authority**: A thread can only perform an action if it possesses a specific **Capability (Cap)** for the target object.
*   **Granularity**: Access rights are fine-grained (e.g., Read-only, Write-only, Grant-only).

## 2. CSpace (Capability Space)

Every process (or protection domain) has a **CSpace**.
*   A CSpace is a table (conceptually) mapping integer indices to Kernel Capabilities.
*   **CPTR (Capability Pointer)**: The integer index used by user-space code to refer to a capability (similar to a file descriptor `fd`).
*   The kernel manages the CSpace; user space cannot forge capabilities.

## 3. Kernel Objects

Capabilities point to specific kernel objects:

| Object Type | Description |
| :--- | :--- |
| **Untyped** | A region of free physical memory. Can be "retyped" into other objects. |
| **TCB** | Thread Control Block. Represents a thread. Invoking it allows suspending/resuming/configuring the thread. |
| **Endpoint** | IPC Port. Used for message passing. |
| **Frame** | A physical memory page (e.g., 4KB). Can be mapped into a VSpace. |
| **PageTable** | A level of the hardware page table structure. |
| **IrqHandler** | Authority to manage a specific hardware interrupt. |
| **CNode** | A node in the CSpace structure (storage for caps). |

## 4. Capability Operations

### 4.1 Invocation
The primary use of a cap.
`syscall_invoke(cptr, args...)`
*   If `cptr` points to an Endpoint -> Send IPC.
*   If `cptr` points to a PageTable -> Map a page.
*   If `cptr` points to a TCB -> Suspend/Resume.

### 4.2 Derivation (Minting)
A thread can create a new capability from an existing one, potentially with reduced rights.
*   **Mint**: Create a copy with a specific **Badge** or reduced permissions (e.g., Read-Write Frame -> Read-Only Frame).

### 4.3 Transfer
Capabilities can be moved between CSpaces via IPC.
*   **Grant**: Move a cap to another process. The sender loses it (or shares it, depending on implementation).
*   This is how a server gives a client access to a shared memory buffer (Granting a Frame cap).

### 4.4 Revocation
*   **Revoke**: A parent can delete all child capabilities derived from a specific capability. This allows reclaiming resources.

## 5. Badges

Badges are a crucial feature for server implementation.
*   A server holds the "Master" Receive Cap for an Endpoint.
*   The server "Mints" a Send Cap with a unique **Badge** (an integer tag) for each client.
*   When Client A sends a message using its Badged Cap, the Server receives the message along with the Badge value.
*   **Result**: The server knows exactly which client sent the message, without the kernel needing a global "Client ID" concept. The Badge *is* the identity.
