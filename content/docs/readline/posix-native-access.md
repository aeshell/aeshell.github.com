---
date: '2026-05-20T12:00:00+02:00'
draft: false
title: 'POSIX Native Access'
weight: 17
---

On POSIX systems (Linux and macOS), Æsh Readline needs access to terminal attributes, size, and raw I/O for interactive line editing. Starting with version 3.8, the `terminal-tty` module ships as a **multi-release JAR** with two implementations:

| Java Version | Implementation | How It Works |
|-------------|----------------|-------------|
| 8 -- 21 | `ExecPty` (subprocess) | Spawns `stty` and `tty` commands |
| 22+ | `FfmPty` (FFM) | Calls `tcgetattr`/`tcsetattr`/`ioctl` directly |

On Java 22+, the Foreign Function & Memory API calls libc directly from pure Java -- no subprocess spawning, no dependency on `stty` being installed.

## What Changed

The `ExecPty` implementation spawns external processes for every terminal operation:

| Operation | ExecPty (stty) | FfmPty (FFM) |
|-----------|---------------|--------------|
| Get terminal attributes | `stty -a` (~5ms) | `tcgetattr()` (~5us) |
| Set terminal attributes | `stty <flags>` (~5ms) | `tcsetattr()` (~5us) |
| Get terminal size | `stty size` (~5ms) | `ioctl(TIOCGWINSZ)` (~5us) |
| Discover TTY device | `tty` command (~5ms) | `open("/dev/tty")` (~5us) |

Terminal initialization, which involves 3-5 subprocess spawns, drops from ~15-25ms to ~25us.

## Runtime Requirements

Applications running on Java 22+ must enable native access for the FFM API:

```
java --enable-native-access=ALL-UNNAMED -jar myapp.jar
```

On Java 8-21, no special flags are needed -- `ExecPty` is used automatically.

## How It Works

The multi-release JAR contains `FfmPty` alongside the existing `ExecPty`:

```
terminal-tty.jar
├── org/aesh/terminal/tty/impl/ExecPty.class                (stty, Java 8)
├── META-INF/versions/22/org/aesh/terminal/tty/impl/FfmPty.class  (FFM, Java 22+)
├── META-INF/versions/22/org/aesh/terminal/tty/impl/LibC.class
├── META-INF/versions/22/org/aesh/terminal/tty/impl/PosixConstants.class
└── META-INF/MANIFEST.MF                                    (Multi-Release: true)
```

`TerminalBuilder` tries `FfmPty` first via `Class.forName()`. If it fails (Java < 22, or FFM initialization error), it falls back to `ExecPty`. No configuration needed.

`FfmPty` implements the same `Pty` interface as `ExecPty`, so `PosixSysTerminal`, `TerminalConnection`, and all downstream code require zero changes.

## Platform Differences

The FFM implementation handles Linux and macOS differences:

| Aspect | Linux | macOS |
|--------|-------|-------|
| termios flag size | 4 bytes (`unsigned int`) | 8 bytes (`unsigned long`) |
| termios struct size | 60 bytes | 72 bytes |
| c_cc array size | 32 entries | 20 entries |
| VMIN index | 6 | 16 |
| VTIME index | 5 | 17 |
| TTY file descriptor | `open("/dev/tty", O_RDWR)` | `STDIN_FILENO` (fd 0) |
| TIOCGWINSZ | `0x5413` | `0x40087468` |

On macOS, `STDIN_FILENO` is used instead of opening `/dev/tty` because `poll()` does not work reliably with a separately opened TTY on macOS.

## Input I/O

`FfmPty` provides an `InputStream` that uses `poll()` + `read()` for terminal input:

- `poll()` with 100ms timeout for responsiveness to close()
- Bulk reads using a 1024-byte native buffer -- reads as many bytes as the kernel provides in a single `read()` syscall
- `read()` retries on EINTR (signal interruption between poll and read)
- POLLHUP/POLLERR detection for EOF
- No reader thread needed for the poll-based path (though the existing `TerminalConnection` read loop still runs in a thread)

The bulk read buffer matches the typical maximum bytes returned by a single PTY master read (1 KiB on macOS, up to 4 KiB on Linux). A 6-byte escape sequence like Ctrl+Up costs 2 syscalls (1 poll + 1 read) rather than 12.

## Fallback Behavior

If `FfmPty` fails to initialize for any reason, `TerminalBuilder` logs a warning and falls back to `ExecPty`:

- `ClassNotFoundException` (Java < 22) -- logged at FINE level
- `IOException` or `UnsupportedOperationException` from `FfmPty.current()` -- logged at FINE level
- Other exceptions (FFM bugs, platform issues) -- logged at WARNING level

This means upgrading the JDK version transparently enables the FFM path with no code changes.

## See Also

- [Windows Native Access](windows-native-access) -- the equivalent MRJAR for Windows Console API
- [Terminal](terminal) -- terminal abstraction and attributes
- [Connection](connection) -- connection lifecycle and handlers
