# squeez — Windows Fork

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-stable-orange.svg)](https://www.rust-lang.org)

> Fork of [claudioemmanuel/squeez](https://github.com/claudioemmanuel/squeez) with native Windows support via `windows-sys`.

Token compression + context optimization for Claude Code. Runs automatically via hooks. No configuration required.

## Motivation

The upstream squeez uses POSIX-only APIs (`libc::kill`, `libc::signal`, `process_group(0)`) that do not compile on Windows. This fork replaces all 7 POSIX calls with Win32 equivalents using conditional compilation (`#[cfg(unix)]` / `#[cfg(windows)]`), preserving full macOS/Linux compatibility while enabling Windows support.

## What it does

- **Bash compression** — intercepts every command, removes noise, up to 95% token reduction
- **Session memory** — injects a summary of prior sessions at session start
- **Token tracking** — tracks context usage across all tool calls
- **Compact warning** — alerts when session approaches context limit (80% of budget)

## Windows Support

### POSIX → Win32 mapping

| POSIX call | Win32 replacement | File |
|-----------|-------------------|------|
| `libc::kill(-pid, SIGTERM)` | `GenerateConsoleCtrlEvent(CTRL_BREAK_EVENT, pid)` | `wrap.rs` |
| `libc::signal(SIGTERM\|SIGINT, handler)` | `SetConsoleCtrlHandler(handler, TRUE)` | `wrap.rs` |
| `CommandExt::process_group(0)` | `CommandExt::creation_flags(CREATE_NEW_PROCESS_GROUP)` | `wrap.rs` |
| `Command::new("sh")` | `Command::new("bash")` | `wrap.rs` |
| `Utc::now()` (chrono) | `std::time::SystemTime` | `session.rs` |

### Key design decision

Windows does not have POSIX signals. The equivalent mechanism uses Console Control Events:
- Child processes are spawned with `CREATE_NEW_PROCESS_GROUP` so they can receive `CTRL_BREAK_EVENT` independently.
- The parent registers a `SetConsoleCtrlHandler` callback that forwards Ctrl+C / Ctrl+Break / Close events to the child's process group via `GenerateConsoleCtrlEvent`.
- `CTRL_BREAK_EVENT` is used instead of `CTRL_C_EVENT` because processes in a new group ignore `CTRL_C_EVENT` by default.

### Dependencies

```toml
[target.'cfg(unix)'.dependencies]
libc = "0.2"

[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.59", features = [
    "Win32_Foundation",
    "Win32_System_Console",
    "Win32_System_Threading",
] }
```

## Install (Windows)

### From source

```bash
git clone https://github.com/Malvin74/squeez.git
cd squeez
cargo build --release
mkdir -p "$HOME/.claude/squeez/bin"
cp target/release/squeez.exe "$HOME/.claude/squeez/bin/squeez.exe"
cp hooks/*.sh "$HOME/.claude/squeez/hooks/"
```

### Hook setup

Add to `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "bash \"$HOME/.claude/squeez/hooks/pretooluse.sh\""
      }]
    }],
    "PostToolUse": [{
      "hooks": [{
        "type": "command",
        "command": "bash \"$HOME/.claude/squeez/hooks/posttooluse.sh\""
      }]
    }],
    "SessionStart": [{
      "hooks": [{
        "type": "command",
        "command": "bash \"$HOME/.claude/squeez/hooks/session-start.sh\""
      }]
    }]
  }
}
```

Restart Claude Code. Squeez is active.

### Important: LF line endings

The hook scripts **must** have LF (Unix) line endings, not CRLF. If you copy them on Windows, fix with:

```bash
sed -i 's/\r$//' ~/.claude/squeez/hooks/*.sh
```

Also replace `python3` with `python` in the scripts if your system only has `python.exe`:

```bash
sed -i 's/python3/python/g' ~/.claude/squeez/hooks/*.sh
```

## Install (macOS / Linux)

```bash
curl -fsSL https://raw.githubusercontent.com/claudioemmanuel/squeez/main/install.sh | sh
```

## Escape hatch

Prefix any command with `--no-squeez` to bypass compression:

```
--no-squeez git log --all --graph
```

## Benchmarks

Measured on macOS (Apple Silicon), token estimate = chars/4. Run with `bash bench/run.sh`.

| Fixture | Before | After | Reduction | Latency |
|---------|--------|-------|-----------|---------|
| `ps aux` | 40,373 tk | 2,352 tk | **-95%** | 6ms |
| `git log` (200 commits) | 2,667 tk | 819 tk | **-70%** | 4ms |
| `find` (deep tree) | 424 tk | 134 tk | **-69%** | 3ms |
| `git status` | 50 tk | 16 tk | **-68%** | 3ms |
| `docker logs` | 665 tk | 186 tk | **-73%** | 5ms |
| `npm install` | 524 tk | 231 tk | **-56%** | 3ms |
| `ls -la` | 1,782 tk | 886 tk | **-51%** | 4ms |
| `git diff` | 502 tk | 317 tk | **-37%** | 4ms |
| `env` dump | 441 tk | 287 tk | **-35%** | 3ms |

## Configuration

Optional `~/.claude/squeez/config.ini` (all fields optional):

```ini
max_lines = 200
dedup_min = 3
git_log_max_commits = 20
docker_logs_max_lines = 100
bypass = docker exec, psql, ssh
compact_threshold_tokens = 160000
memory_retention_days = 30
```

## How it works

Three Claude Code hooks work together:

1. **Compression** (`PreToolUse`): Every Bash call is rewritten — `dotnet test` becomes `squeez wrap 'dotnet test'`. The wrap command runs the child process, captures stdout+stderr, applies 4 strategies (smart_filter → dedup → grouping → truncation), and prints a compressed result with a savings header.

2. **Session memory** (`SessionStart`): On each new session, `squeez init` finalizes the previous session summary (files touched, errors resolved, test results, git events) and prints a memory banner.

3. **Token tracking** (`PostToolUse`): Every tool call's output size is tracked. When cumulative tokens cross 80% of context budget, a compact warning is emitted.

## Upstream

This fork tracks [claudioemmanuel/squeez](https://github.com/claudioemmanuel/squeez) as `upstream` remote. Changes are limited to platform abstraction — no upstream features were removed or altered.

## License

[MIT](LICENSE)
