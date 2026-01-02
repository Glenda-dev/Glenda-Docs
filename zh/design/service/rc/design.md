# Rc Shell 设计文档

## 1. 简介
**Rc** 是 Glenda 的默认命令行 Shell。它是 Plan 9 `rc` shell 的移植/重新实现，适应了 Glenda 的微内核架构。它是与 Factotum（进程控制）和 Gopher（文件操作）交互的主要用户界面。

## 2. 设计理念
*   **简单性**: 类似 C 的语法，但更清晰（没有 `sh` 的怪癖）。
*   **一切皆文件**: 深度利用 Gopher 的命名空间。
*   **Capability 感知**: 隐式理解 Glenda 的 capability 模型（例如，通过环境或继承传递 capabilities）。

## 3. 特性

### 3.1 语法
Rc 使用受 C 启发的语法：
*   **变量**: `$var` (字符串列表)。
*   **引用**: 仅 `'单引号'`。
*   **控制流**: `if`, `while`, `for`, `switch`。
*   **函数**: `fn name { commands }`。

### 3.2 进程控制
Rc 与 **Factotum** 交互以管理进程：
*   **执行**: `cmd args` 转换为 `Factotum::SPAWN`。
*   **管道**: `cmd1 | cmd2` 通过 Gopher 创建管道，生成两个进程，并连接它们的 FD。
*   **后台**: `cmd &` 异步生成。

### 3.3 重定向
Rc 与 **Gopher** 交互以进行 I/O 重定向：
*   `> file`: 打开文件用于写入（截断）。
*   `>> file`: 打开文件用于追加。
*   `< file`: 打开文件用于读取。
*   `<> file`: 打开用于读/写。
*   `>[fd]`: 重定向特定 FD。

### 3.4 命名空间操作
Rc 将 Gopher 的命名空间操作暴露为内置命令：
*   `mount`: 挂载服务。
*   `bind`: 绑定目录。
*   `unmount`: 卸载。
*   `rfork`: 创建新命名空间（调用带有 `RFNOMNT` 或类似系统调用的 `Factotum::SPAWN`）。

## 4. 架构

### 4.1 组件
*   **词法分析器/解析器**: 解析 Rc 脚本/命令行。
*   **执行器**: 遍历 AST 并执行命令。
*   **内置命令**: 内部命令 (`cd`, `echo`, `exit`, `mount`, `bind`)。
*   **作业控制**: 跟踪后台进程（Factotum 返回的 PID）。

### 4.2 与系统服务的交互
*   **Factotum**:
    *   `SPAWN`: 启动程序。
    *   `WAIT`: 等待子进程。
*   **Gopher**:
    *   `OPEN/CREATE`: 用于重定向。
    *   `PIPE`: 创建管道。
    *   `MOUNT/BIND`: 命名空间操作。
*   **Tux**:
    *   Rc 本身是一个类 POSIX 应用程序，可能链接到 `libglenda` 或与 Tux/Factotum 对话的 libc。

## 5. 示例脚本

```rc
#!/bin/rc

fn hello {
    echo Hello, $1
}

# 命名空间设置
mount /srv/tcp /net
bind -b /usr/bin /bin

# 管道
ls -l /bin | grep rc

# 后台任务
cp largefile /tmp &

# 控制流
if(~ $#* 0) {
    echo Usage: $0 name
    exit 1
}

hello $1
```

## 6. 实现计划
1.  **核心**: 词法分析器，解析器，AST。
2.  **执行**: 简单命令执行 (Factotum SPAWN)。
3.  **I/O**: 重定向和管道 (Gopher 集成)。
4.  **变量与控制流**: 完整的语言支持。
5.  **内置命令**: `cd`, `mount`, `bind`。
