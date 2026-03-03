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
    ├── main.rs       # 入口点，参数解析与子命令分发
    ├── arch.rs       # 架构相关配置 (如 RISC-V 64)
    ├── build/        # 构建逻辑实现
    │   ├── mod.rs    # 核心构建流水线 (内核, 服务, 库)
    │   ├── cargo.rs  # Rust 项目构建封装
    │   ├── cmake.rs  # C/C++ 项目构建封装
    │   ├── make.rs   # Makefile 项目构建封装
    │   └── image.rs  # 镜像生成 (ISO/IMG/FAT32)
    ├── qemu.rs       # QEMU 运行配置、DTB/ACPI 导出与调试支持
    ├── config.rs     # TOML 配置文件解析 (如 config.toml, hello.toml)
    ├── check.rs      # 代码检查与静态分析工具链
    └── util.rs       # 辅助工具 (objdump, size)
```

## 3. 主要功能

### 3.1 核心构建 (Build)

`cargo xtask build` 命令实现了多语言混合构建流水线：
1.  **环境检查**: 验证工具链（如 `rust-src`, `llvm-tools`）及交叉编译器。
2.  **构建内核**: 调用 `cargo build --package kernel`。
3.  **构建服务**: 自动扫描 `service/` 并在 `target/fsroot/` 下编排。
4.  **构建库**: 构建 `libglenda-rs`、`libape`、`libglenda`。
5.  **C/C++ 支持**: 通过 `cmake` 和 `make` 模块支持移植的 C 库（如 `musl-glenda`）。

### 3.2 运行与调试 (Run / Gdb)

`cargo xtask run` 命令负责：
1.  执行增量构建。
2.  **镜像编排**: 根据 `config.toml` 将内核和服务打包成引导镜像。
3.  **QEMU 自动化**: 
    - 自动配置机器类型（如 `virt`）。
    - 映射磁盘镜像、网络后端等。
    - 支持 `--timeout` 自动结束运行。

`cargo xtask gdb` 命令负责：
1. 启动 QEMU 并进入 `-S -s` (暂停待调) 模式。
2. 提供调试监听端口（默认 1234）。

### 3.3 镜像生成 (Image)

`cargo xtask image` 支持多种输出格式：
- **ISO**: 生成基于 GRUB/Limine 的可引导光盘镜像。
- **IMG**: 生成原始裸磁盘镜像。
- **FAT32/ESP**: 生成包含 EFI 分区的文件系统镜像。

### 3.4 开发辅助 (Util)

- `cargo xtask objdump`: 使用 `llvm-objdump` 反汇编内核并关联源代码。
- `cargo xtask size`: 精确统计内核各段 (`.text`, `.data`, `.bss`) 的空间占用。
- `cargo xtask check`: 封装 `cargo check`，支持对所有工作区成员（内核、库、服务）进行并行类型检查。
- `cargo xtask dump-dtb / dump-acpi`: 从运行中的 QEMU 实例导出硬件描述信息以供驱动开发参考。

## 4. 实现细节

### 4.1 配置文件驱动
`xtask` 高度依赖 `config.toml`。它不再硬编码服务列表，而是动态解析配置中的 `[services]` 和 `[kernel]` 字段，从而支持 `hello.toml` 等不同的引导场景。

### 4.2 错误处理
使用 `anyhow` 提供详细的错误链追踪，确保在构建失败（如 C 库编译报错）时能准确定位。

### 4.3 目录管理
所有构建产物均严格限制在项目根目录下的 `target/` 目录中，通过 `manifest_dir` 准确定位工作区根路径。

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
