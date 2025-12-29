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
| **Untyped** | The root of all physical memory. Represents a contiguous block of free memory. Can be "retyped" into specific kernel objects. |
| **TCB** | Thread Control Block. Represents a thread. Invoking it allows suspending/resuming/configuring the thread. |
| **Endpoint** | IPC Port. Used for message passing. |
| **Frame** | A physical memory page (e.g., 4KB). Can be mapped into a VSpace. |
| **PageTable** | A level of the hardware page table structure. |
| **IrqHandler** | Authority to manage a specific hardware interrupt. |
| **CNode** | A node in the CSpace structure (storage for caps). |
| **Reply** | A special, one-shot capability used to reply to a `Call` invocation. |

## 4. Capability Operations

### 4.1 Invocation
The primary use of a cap is **Invocation** via `sys_invoke`.
*   The specific methods available depend on the object type.
*   See [System Call Interface](syscall.md) for a complete list of methods for each object type.

### 4.2 Retyping (Object Creation)
The **Retype** operation is the only mechanism for creating new kernel objects. It transforms a portion of an `Untyped` memory region into one or more specific kernel objects.

*   **Mechanism**:
    1.  The user invokes the `Retype` method on an **Untyped** capability.
    2.  The kernel verifies that the requested memory range is within the `Untyped` region and is currently free.
    3.  The kernel carves out the memory, initializes the new objects (e.g., zeroing a PageTable), and creates the corresponding capabilities.
    4.  The new capabilities are inserted into the user's CSpace.
*   **Derivation**: The new objects are considered "children" of the original `Untyped` capability in the **Capability Derivation Tree (CDT)**.

### 4.3 Derivation (Minting)
A thread can create a new capability from an existing one, potentially with reduced rights.
*   **Mint**: Create a copy with a specific **Badge** or reduced permissions (e.g., Read-Write Frame -> Read-Only Frame).
*   This is performed by invoking the `Mint` method on a **CNode** capability.

### 4.4 Transfer (Delegation)
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

### 4.5 Revocation & Resource Reclamation
Glenda ensures memory safety and resource accounting through a combination of **Reference Counting** and the **Capability Derivation Tree (CDT)**.

*   **Reference Counting**: Every kernel object (TCB, CNode, Endpoint, etc.) is reference-counted.
    *   **Capability Ownership**: In the Rust kernel, the `Capability` struct implements `Clone` and `Drop`.
    *   **Increment**: Cloning a `Capability` (via `Clone` or when passing to methods like `insert`) increments the underlying object's reference count.
    *   **Decrement**: Dropping a `Capability` (via `Drop` or when a CNode slot is overwritten) decrements the count.
    *   **Destruction**: When the count reaches zero, the object is destroyed. If the object was created from `Untyped` memory, that memory is logically returned to the parent `Untyped` region or marked as free for future retyping.
*   **Capability Derivation Tree (CDT)**: The CDT tracks the "parent-child" relationship between capabilities.
    *   When an `Untyped` region is retyped, the resulting objects are children of that `Untyped` capability.
    *   When a capability is `Minted`, the new capability is a child of the original.
*   **Revoke**: The `Revoke` operation allows a user to recursively delete all capabilities derived from a target capability.
    *   If `Revoke` is called on an `Untyped` capability, all objects created from that memory are destroyed, and the memory is reclaimed.
    *   This ensures that a resource provider can always take back resources it has delegated to a child process.

## 5. Badges

Badges are a crucial feature for server implementation.
*   A server holds the "Master" Receive Cap for an Endpoint.
*   The server "Mints" a Send Cap with a unique **Badge** (an integer tag) for each client.
*   **Immutable Identity**: Once a capability is badged, its badge cannot be modified. This ensures that a client cannot forge its identity by re-badging a capability it received.
*   **Automatic Delivery**: When a thread sends a message using a badged capability, the kernel automatically extracts the badge and delivers it to the receiver (typically via a designated register like `t0`).
*   **Result**: The server knows exactly which client sent the message, without the kernel needing a global "Client ID" concept. The Badge *is* the identity.
