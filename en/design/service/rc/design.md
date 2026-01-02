# Rc Shell Design Document

## 1. Introduction
**Rc** is the default command-line shell for Glenda. It is a port/re-implementation of the Plan 9 `rc` shell, adapted for Glenda's microkernel architecture. It serves as the primary user interface for interacting with Factotum (process control) and Gopher (file operations).

## 2. Design Philosophy
*   **Simplicity**: C-like syntax but cleaner (no `sh` quirks).
*   **Everything is a File**: Leverages Gopher's namespace heavily.
*   **Capability Aware**: Understands Glenda's capability model implicitly (e.g., passing capabilities via environment or inheritance).

## 3. Features

### 3.1 Syntax
Rc uses a C-inspired syntax:
*   **Variables**: `$var` (list of strings).
*   **Quoting**: `'single quotes'` only.
*   **Control Flow**: `if`, `while`, `for`, `switch`.
*   **Functions**: `fn name { commands }`.

### 3.2 Process Control
Rc interacts with **Factotum** to manage processes:
*   **Execution**: `cmd args` translates to `Factotum::SPAWN`.
*   **Pipelines**: `cmd1 | cmd2` creates a pipe via Gopher, spawns both processes, and connects their FDs.
*   **Background**: `cmd &` spawns asynchronously.

### 3.3 Redirection
Rc interacts with **Gopher** for I/O redirection:
*   `> file`: Open file for write (truncate).
*   `>> file`: Open file for append.
*   `< file`: Open file for read.
*   `<> file`: Open for read/write.
*   `>[fd]`: Redirect specific FD.

### 3.4 Namespace Manipulation
Rc exposes Gopher's namespace operations as built-in commands:
*   `mount`: Mount a service.
*   `bind`: Bind a directory.
*   `unmount`: Unmount.
*   `rfork`: Create a new namespace (calls `Factotum::SPAWN` with `RFNOMNT` or similar syscall).

## 4. Architecture

### 4.1 Components
*   **Lexer/Parser**: Parses the Rc script/command line.
*   **Executor**: Walks the AST and executes commands.
*   **Builtins**: Internal commands (`cd`, `echo`, `exit`, `mount`, `bind`).
*   **Job Control**: Tracks background processes (PIDs returned by Factotum).

### 4.2 Interaction with System Services
*   **Factotum**:
    *   `SPAWN`: Launch programs.
    *   `WAIT`: Wait for child processes.
*   **Gopher**:
    *   `OPEN/CREATE`: For redirections.
    *   `PIPE`: Create pipes.
    *   `MOUNT/BIND`: Namespace operations.
*   **Tux**:
    *   Rc itself is a POSIX-like application, likely linked against `libglenda` or a libc that talks to Tux/Factotum.

## 5. Example Script

```rc
#!/bin/rc

fn hello {
    echo Hello, $1
}

# Namespace setup
mount /srv/tcp /net
bind -b /usr/bin /bin

# Pipeline
ls -l /bin | grep rc

# Background task
cp largefile /tmp &

# Control flow
if(~ $#* 0) {
    echo Usage: $0 name
    exit 1
}

hello $1
```

## 6. Implementation Plan
1.  **Core**: Lexer, Parser, AST.
2.  **Execution**: Simple command execution (Factotum SPAWN).
3.  **I/O**: Redirection and Pipes (Gopher integration).
4.  **Variables & Control Flow**: Full language support.
5.  **Builtins**: `cd`, `mount`, `bind`.
