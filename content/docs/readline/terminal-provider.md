---
date: '2026-06-10T15:00:00+01:00'
draft: false
title: 'Terminal Provider SPI'
weight: 30
---

A `ServiceLoader`-based SPI for pluggable terminal implementations, allowing different terminal backends to be discovered and loaded at runtime.

## How It Works

`TerminalBuilder.build()` discovers all `TerminalProvider` implementations via `ServiceLoader`, filters by `isSupported()`, sorts by priority (highest first), and tries each until one succeeds. If none works, it falls back to `ExternalTerminal`.

## Built-in Providers

| Provider | Priority | Platform | Description |
|----------|----------|----------|-------------|
| `FfmTerminalProvider` | 100 | POSIX, Java 22+ | FFM-based PTY (best performance) |
| `WinSysTerminalProvider` | 100 | Windows | Windows console (JNI or FFM) |
| `CygwinTerminalProvider` | 75 | Cygwin/MSYS2 | POSIX PTY via Cygwin |
| `ExecPtyTerminalProvider` | 50 | POSIX | exec-based PTY (stty/tty) |

## Custom Provider

Implement `TerminalProvider` and register via `META-INF/services`:

```java
public class MyTerminalProvider implements TerminalProvider {
    @Override
    public String name() { return "my-terminal"; }

    @Override
    public boolean isSupported() {
        return /* check platform/capabilities */;
    }

    @Override
    public int priority() { return 110; } // higher than built-in

    @Override
    public Terminal createTerminal(String name, String type,
            boolean nativeSignals) throws IOException {
        return new MyTerminal(name, nativeSignals);
    }
}
```

Register in `META-INF/services/org.aesh.terminal.provider.TerminalProvider`:
```
com.example.MyTerminalProvider
```

## FFM Provider and Native Access

The `FfmTerminalProvider` uses Java's Foreign Function & Memory (FFM) API to call POSIX terminal functions directly, avoiding subprocess spawning. It requires:

- **Java 22+** — FFM was finalized in JEP 454
- **`--enable-native-access=ALL-UNNAMED`** — required JVM flag for classpath usage

When `--enable-native-access` is **not set**, the FFM provider silently reports `isSupported() = false` and the fallback to `ExecPtyTerminalProvider` happens automatically — **no JVM warning is emitted**. This is intentional: a library should not trigger restricted method warnings when it has a working fallback.

To verify which provider is active, check the log output at `FINE` level:

```
FINE: Using FFM-based PTY          (FfmTerminalProvider selected)
FINE: Using exec-based PTY         (ExecPtyTerminalProvider fallback)
```

## Piped and Redirected I/O

When stdin is not a terminal (piped or redirected from a file), `TerminalBuilder` detects this via `isatty(STDIN_FILENO)` and goes directly to `ExternalTerminal` without trying system terminal providers. This follows standard POSIX convention.

```bash
# All of these use ExternalTerminal automatically:
echo "help" | java -cp ... MyApp
java -cp ... MyApp < commands.txt
cat commands.txt | java -cp ... MyApp
```

When stdout is not a terminal (redirected to a file or piped), ANSI escape codes are automatically suppressed — `supportsAnsi()` returns `false`. This matches the behavior of `ls --color=auto`, `grep --color=auto`, and other POSIX programs.

```bash
# ANSI output suppressed in output.txt:
java -cp ... MyApp > output.txt

# Interactive input, but no ANSI in pipe:
java -cp ... MyApp | tee log.txt
```

## Priority

Higher priority values are preferred. When multiple providers are supported, the one with the highest priority is tried first. If it fails, the next is tried.

Recommended ranges:
- **100+** — native/FFM providers (best performance)
- **50-99** — process-based providers (exec/JNI)
- **1-49** — fallback providers (dumb terminals)
