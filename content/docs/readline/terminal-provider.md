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

## Priority

Higher priority values are preferred. When multiple providers are supported, the one with the highest priority is tried first. If it fails, the next is tried.

Recommended ranges:
- **100+** — native/FFM providers (best performance)
- **50-99** — process-based providers (exec/JNI)
- **1-49** — fallback providers (dumb terminals)
