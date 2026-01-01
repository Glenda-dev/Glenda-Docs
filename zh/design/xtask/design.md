# Xtask: 构建与开发工具

`xtask` 是 Glenda 项目的构建系统和开发工具，使用 Rust 编写。它利用 Cargo 的 `xtask` 模式来管理复杂的构建流程、镜像生成和运行任务。

## 1. 设计目标

- **跨平台**: 统一 Linux, macOS 和 Windows 上的开发体验。
- **自动化**: 自动化构建内核、用户态服务、生成文件系统镜像等任务。
- **易用性**: 提供简单的命令行接口。

## 2. 目录结构

```
xtask/
├── Cargo.toml
└── src/
    ├── main.rs       # 入口点，参数解析
    ├── build.rs      # 构建逻辑 (内核, 服务)
    ├── qemu.rs       # QEMU 运行配置
    ├── fs.rs         # 文件系统镜像生成
    └── config.rs     # 构建配置
```

## 3. 主要功能

### 3.1 构建 (Build)

`cargo xtask build` 命令负责：
1.  **构建内核**: 调用 `cargo build --package kernel --target ...`。
2.  **构建服务**: 遍历 `service/` 目录，构建每个服务。
3.  **构建库**: 构建 `libglenda-rs`。
4.  **生成二进制文件**: 使用 `objcopy` 将 ELF 文件转换为二进制格式（如果需要）。

### 3.2 运行 (Run)

`cargo xtask run` 命令负责：
1.  执行构建步骤。
2.  **生成镜像**: 将内核和服务打包成系统镜像（如包含 Bootloader 的镜像）。
3.  **启动 QEMU**: 使用正确的参数启动 QEMU。

参数：
- `--smp <N>`: CPU 核心数。
- `--debug`: 启用 QEMU 调试监听 (GDB)。
- `--log <LEVEL>`: 设置日志级别。

### 3.3 文件系统 (FS)

`cargo xtask fs` 命令负责：
1.  创建目录结构。
2.  将编译好的服务二进制文件复制到文件系统目录。
3.  生成文件系统镜像（如 RomFS 或 FAT32）。

### 3.4 辅助工具

- `cargo xtask asm`: 反汇编内核。
- `cargo xtask size`: 显示内核大小。
- `cargo xtask gdb`: 启动 GDB 并连接到 QEMU。

## 4. 实现细节

### 4.1 依赖管理
`xtask` 依赖于 `clap` 进行参数解析，`anyhow` 进行错误处理，`xshell` 或 `std::process::Command` 执行 shell 命令。

### 4.2 配置文件
构建配置（如目标架构、QEMU 路径）可以从 `config.toml` 读取，也可以使用默认值。

## 5. 使用示例

```bash
# 构建并运行
cargo xtask run

# 仅构建内核
cargo xtask build --kernel

# 启用调试模式运行
cargo xtask run --debug

# 生成文档
cargo xtask doc
```
