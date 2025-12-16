# Build System Design (Xtask)

## 1. Overview

Glenda uses a custom build system based on Rust's `xtask` pattern. This allows us to orchestrate complex build flows involving multiple architectures (kernel in S-Mode, apps in U-Mode) and languages (Rust, C) using standard Rust tooling.

The core responsibility of the build system in the microkernel architecture is to:
1.  Compile the Kernel.
2.  Compile User-Space Components (Root Task, Drivers, Servers).
3.  Package components into a **Boot Payload**.
4.  Embed the payload into the Kernel image.

## 2. Configuration (`components.toml`)

The build system is data-driven. The list of user-space components to be packaged is defined in `components.toml` at the workspace root.

```toml
[[components]]
name = "root-task"
path = "service/hello"           # Source directory
build_cmd = "make"               # Command to build the component
output_bin = "service/hello/hello.bin" # Path to the resulting binary
kind = "root_task"               # Type: root_task, driver, server, file
```

## 3. Build Flow

When running `cargo xtask build`, the following steps occur:

1.  **Parse Configuration**: `xtask` reads `components.toml`.
2.  **Build Components**:
    *   Iterates through each component.
    *   Executes the specified `build_cmd` in the component's directory.
    *   Verifies the existence of `output_bin`.
3.  **Generate Payload (`modules.bin`)**:
    *   Collects all component binaries.
    *   Generates a **Boot Module Header** (Magic + Count).
    *   Generates **Metadata Entries** for each component (Type, Offset, Size, Name).
    *   Concatenates Header + Metadata + Binary Data into a single `target/modules.bin` file.
4.  **Generate Rust Source (`proc_payload.rs`)**:
    *   Creates a Rust source file that uses `include_bytes!` to embed `modules.bin`.
    *   Defines constants like `BOOT_MODULES_BLOB`.
5.  **Build Kernel**:
    *   Compiles the kernel crate.
    *   The kernel includes `proc_payload.rs` during compilation, effectively embedding the user-space binaries into its `.rodata` section.

## 4. Payload Format

The `modules.bin` blob has the following layout:

| Offset | Size | Field | Description |
| :--- | :--- | :--- | :--- |
| 0 | 4 | `magic` | `0xGLENDA01` (Little Endian) |
| 4 | 4 | `count` | Number of modules |
| 8 | 48 | `Entry 0` | Metadata for first module |
| ... | ... | ... | ... |
| 8 + N*48 | ... | `Data` | Concatenated binary data |

**Entry Structure (48 bytes):**
*   `type` (1 byte): 0=RootTask, 1=Driver, 2=Server, 3=File
*   `offset` (4 bytes): Offset of data relative to start of blob.
*   `size` (4 bytes): Size of data in bytes.
*   `name` (32 bytes): UTF-8 string, null-padded.
*   `padding` (7 bytes): Reserved.

## 6. Implementation Steps

### Step 1: Add Dependencies
Update `xtask/Cargo.toml` to include `serde` and `toml` for configuration parsing.

### Step 2: Create Configuration
Create `components.toml` in the workspace root with the initial Root Task configuration.

### Step 3: Refactor `xtask`
1.  Define `Config` and `ComponentConfig` structs deriving `Deserialize`.
2.  Implement `process_components` function to:
    *   Read `components.toml`.
    *   Execute build commands.
    *   Generate `target/bootfs/payload.bin` with the specified header and entry format.
3.  Update `build` command to call `process_components` before building the kernel.

### Step 4: Kernel Integration
1.  Define `BootModuleHeader` and `BootModuleEntry` structs in `kernel/src/bootinfo.rs` (matching the binary format).
2.  Create `kernel/src/asm/payload.S` to `.incbin` the payload.
3.  Create `kernel/src/boot_payload.rs` to expose the payload slice.
4.  Implement a parser in the kernel to iterate over the modules blob and identify the Root Task.

