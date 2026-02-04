# FS (File System) 协议定义

## Protocol Label Range
`0x500 - 0x5FF`

## 描述
FS 协议定义了文件系统的操作交互。它涵盖了命名空间操作（如打开、创建目录）和文件句柄操作（如读、写）。

此协议ID常量定义在 `PROTOCOL_ID = 0x500`。

## 操作指令

### 命名空间操作 (Namespace Operations)
这些操作通常在根目录或当前工作目录的能力（Capability）上调用。

| ID | 名称 | 参数 (Registers) | 额外输入 (Buffer/Caps) | 返回 (Registers) | 额外返回 (Buffer/Caps) |
|---|---|---|---|---|---|
| 1 | **OPEN** | `flags`, `mode` | String: `path` | - | Cap: `handle` |
| 2 | **MKDIR** | `mode` | String: `path` | - | - |
| 3 | **UNLINK** | - | String: `path` | - | - |
| 4 | **RENAME** | - | String: `old_path` \| `new_path`* | - | - |
| 5 | **STAT_PATH**| - | String: `path` | - | Struct: `Stat` |

\* `RENAME` 操作中，`old_path` 和 `new_path` 需顺序打包在 IPC 缓冲区中。

### 文件句柄操作 (File Handle Operations)
这些操作在已打开的文件句柄能力上调用。

| ID | 名称 | 参数 (Registers) | 额外输入 (Buffer) | 返回 (Registers) | 额外返回 (Buffer) |
|---|---|---|---|---|---|
| 10 | **READ** | `size`, `offset` | - | - | Bytes |
| 11 | **WRITE** | `size`, `offset` | Bytes | `written` | - |
| 12 | **CLOSE** | - | - | - | - |
| 13 | **STAT** | - | - | - | Struct: `Stat` |
| 14 | **GETDENTS**| `count` | - | - | List: `DEntry` |
| 15 | **SEEK** | `offset`, `whence` | - | `new_offset` | - |
| 16 | **SYNC** | - | - | - | - |
| 17 | **TRUNCATE**| `size` | - | - | - |

## 数据结构

所有的结构体都应标记为 `#[repr(C)]` 以保证内存布局一致。

### Struct Stat

| 字段 | 类型 | 描述 |
|---|---|---|
| dev | u64 | 设备ID |
| ino | u64 | Inode号 |
| mode | u32 | 文件模式/权限 (见 `FileType`) |
| nlink | u32 | 硬链接数 |
| uid | u32 | 用户ID |
| gid | u32 | 组ID |
| size | u64 | 文件大小 |
| atime_sec | i64 | 访问时间 (秒) |
| atime_nsec | i64 | 访问时间 (纳秒) |
| mtime_sec | i64 | 修改时间 (秒) |
| mtime_nsec | i64 | 修改时间 (纳秒) |
| ctime_sec | i64 | 状态改变时间 (秒) |
| ctime_nsec | i64 | 状态改变时间 (纳秒) |
| blksize | i64 | 块大小 |
| blocks | i64 | 块数量 |

### Struct DEntry

用于 `GETDENTS` 调用返回目录项。

| 字段 | 类型 | 描述 |
|---|---|---|
| d_ino | u64 | Inode号 |
| d_off | i64 | 目录偏移量 |
| d_reclen | u16 | 记录长度 |
| d_type | u8 | 文件类型 |
| d_name | [u8; 256] | 文件名 (C string, padded with 0) |

## 常量定义

### OpenFlags (Bitflags)
用于 `OPEN` 操作的 flags 参数。

| Flag | Octal | Hex | Description |
|---|---|---|---|
| O_RDONLY | 0o0 | 0x0 | 只读 |
| O_WRONLY | 0o1 | 0x1 | 只写 |
| O_RDWR | 0o2 | 0x2 | 读写 |
| O_CREAT | 0o100 | 0x40 | 创建文件 |
| O_EXCL | 0o200 | 0x80 | 排他创建 |
| O_TRUNC | 0o1000 | 0x200 | 截断文件 |
| O_APPEND | 0o2000 | 0x400 | 追加写入 |
| O_DIRECTORY| 0o200000 | 0x10000 | 必须是目录 |

### FileType (Bitflags)
用于 `Stat.mode` 字段。

| Flag | Octal | Description |
|---|---|---|
| S_IFMT | 0o170000 | 类型掩码 |
| S_IFDIR | 0o040000 | 目录 |
| S_IFREG | 0o100000 | 常规文件 |

### Seek Whence
用于 `SEEK` 操作的 whence 参数。

- `SEEK_SET` (0): 从文件头开始
- `SEEK_CUR` (1): 从当前位置开始
- `SEEK_END` (2): 从文件尾开始
