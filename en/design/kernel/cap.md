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
The primary use of a cap is **Invocation** via `sys_invoke`.
*   The specific methods available depend on the object type.
*   See [System Call Interface](syscall.md) for a complete list of methods for each object type.

### 4.2 Derivation (Minting)
A thread can create a new capability from an existing one, potentially with reduced rights.
*   **Mint**: Create a copy with a specific **Badge** or reduced permissions (e.g., Read-Write Frame -> Read-Only Frame).
*   This is performed by invoking the `Mint` method on a **CNode** capability.

### 4.3 Transfer (Delegation)
Capabilities can be transferred between CSpaces via IPC messages. This is the primary mechanism for authority delegation.

*   **Mechanism**:
    1.  **Sender**:
        *   Sets the `HasCap` flag in the `MsgTag`.
        *   Places the CPTR of the capability to be transferred into `utcb.cap_transfer`.
        *   Must have the **`Grant`** right on the capability being transferred.
    2.  **Receiver**: Specifies a "Receive Window" (a CPTR to a slot) in its own `utcb.recv_window` where the incoming capability should be stored.
    3.  **Kernel**:
        *   During the IPC rendezvous, the kernel looks up the capability in the sender's CSpace.
        *   It verifies the `Grant` right on the capability.
        *   It inserts the capability into the receiver's CSpace at the specified slot.
*   **Grant**: The standard operation is to copy the capability. The sender retains access unless they explicitly delete it.
*   **Use Case**: A server grants a client access to a shared memory buffer by sending a `Frame` cap.

### 4.4 Revocation & Resource Reclamation
*   **Reference Counting**: Glenda uses atomic reference counting for kernel objects (`TCB`, `Endpoint`, `CNode`).
    *   Each object has a `ref_count` field.
    *   Creating a new Capability (Mint/Copy) increments the count.
    *   Deleting a Capability (Delete/Revoke) decrements the count.
    *   When `ref_count` reaches 0, the object is logically destroyed (e.g., removed from scheduler queues).
*   **Revoke**: A parent can delete all child capabilities derived from a specific capability. This allows reclaiming resources.
*   This is performed by invoking the `Revoke` method on a **CNode** capability.

## 5. Badges

Badges are a crucial feature for server implementation.
*   A server holds the "Master" Receive Cap for an Endpoint.
*   The server "Mints" a Send Cap with a unique **Badge** (an integer tag) for each client.
*   When Client A sends a message using its Badged Cap, the Server receives the message along with the Badge value.
*   **Result**: The server knows exactly which client sent the message, without the kernel needing a global "Client ID" concept. The Badge *is* the identity.
