# FS (File System) Protocol Definition

## Protocol Label Range
`0x500 - 0x5FF`

## Description
The FS protocol defines file system operation interactions. It covers both namespace operations (such as open, mkdir) and file handle operations (such as read, write).

The protocol ID constant is defined at `PROTOCOL_ID = 0x500`.

## Operations

### Namespace Operations
These operations are usually invoked on the root directory or current working directory Capability.

| ID | Name | Arguments (Registers) | Extra Input (Buffer/Caps) | Returns (Registers) | Extra Returns (Buffer/Caps) |
|---|---|---|---|---|---|
| 1 | **OPEN** | `flags`, `mode` | String: `path` | - | Cap: `handle` |
| 2 | **MKDIR** | `mode` | String: `path` | - | - |
| 3 | **UNLINK** | - | String: `path` | - | - |
| 4 | **RENAME** | - | String: `old_path` \| `new_path`* | - | - |
| 5 | **STAT_PATH**| - | String: `path` | - | Struct: `Stat` |

\* In `RENAME` operation, `old_path` and `new_path` need to be packed sequentially in the IPC buffer.

### File Handle Operations
These operations are invoked on an open file handle capability.

| ID | Name | Arguments (Registers) | Extra Input (Buffer) | Returns (Registers) | Extra Returns (Buffer) |
|---|---|---|---|---|---|
| 10 | **READ** | `size`, `offset` | - | - | Bytes |
| 11 | **WRITE** | `size`, `offset` | Bytes | `written` | - |
| 12 | **CLOSE** | - | - | - | - |
| 13 | **STAT** | - | - | - | Struct: `Stat` |
| 14 | **GETDENTS**| `count` | - | - | List: `DEntry` |
| 15 | **SEEK** | `offset`, `whence` | - | `new_offset` | - |
| 16 | **SYNC** | - | - | - | - |
| 17 | **TRUNCATE**| `size` | - | - | - |

## Data Structures

All structures should be marked with `#[repr(C)]` to ensure consistent memory layout.

### Struct Stat

| Field | Type | Description |
|---|---|---|
| dev | u64 | Device ID |
| ino | u64 | Inode number |
| mode | u32 | File mode/permissions (see `FileType`) |
| nlink | u32 | Number of hard links |
| uid | u32 | User ID |
| gid | u32 | Group ID |
| size | u64 | File size |
| atime_sec | i64 | Access time (seconds) |
| atime_nsec | i64 | Access time (nanoseconds) |
| mtime_sec | i64 | Modification time (seconds) |
| mtime_nsec | i64 | Modification time (nanoseconds) |
| ctime_sec | i64 | Status change time (seconds) |
| ctime_nsec | i64 | Status change time (nanoseconds) |
| blksize | i64 | Block size |
| blocks | i64 | Number of blocks |

### Struct DEntry

Used in `GETDENTS` call to return directory entries.

| Field | Type | Description |
|---|---|---|
| d_ino | u64 | Inode number |
| d_off | i64 | Directory offset |
| d_reclen | u16 | Record length |
| d_type | u8 | File type |
| d_name | [u8; 256] | File name (C string, padded with 0) |

## Constant Definitions

### OpenFlags (Bitflags)
Used for the `flags` argument in `OPEN` operation.

| Flag | Octal | Hex | Description |
|---|---|---|---|
| O_RDONLY | 0o0 | 0x0 | Read-only |
| O_WRONLY | 0o1 | 0x1 | Write-only |
| O_RDWR | 0o2 | 0x2 | Read-write |
| O_CREAT | 0o100 | 0x40 | Create file |
| O_EXCL | 0o200 | 0x80 | Exclusive creation |
| O_TRUNC | 0o1000 | 0x200 | Truncate file |
| O_APPEND | 0o2000 | 0x400 | Append |
| O_DIRECTORY| 0o200000 | 0x10000 | Must be directory |

### FileType (Bitflags)
Used for `Stat.mode` field.

| Flag | Octal | Description |
|---|---|---|
| S_IFMT | 0o170000 | Type mask |
| S_IFDIR | 0o040000 | Directory |
| S_IFREG | 0o100000 | Regular file |

### Seek Whence
Used for the `whence` argument in `SEEK` operation.

- `SEEK_SET` (0): Set position relative to the beginning of the file
- `SEEK_CUR` (1): Set position relative to the current position
- `SEEK_END` (2): Set position relative to the end of the file
