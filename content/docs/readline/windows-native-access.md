---
date: '2026-04-18T12:00:00+02:00'
draft: false
title: 'Windows Native Access'
weight: 16
---

On Windows, Æsh Readline needs access to the Windows Console API (Kernel32) for raw terminal input, console mode control, and terminal size detection. The `terminal-tty` module ships as a **multi-release JAR** with two implementations:

| Java Version | Implementation | Native Code Required |
|-------------|----------------|---------------------|
| 8 -- 21 | JNI (`aesh-console.dll`) | Yes |
| 22+ | FFM (`java.lang.foreign`) | No |

On Java 22+, the Foreign Function & Memory API calls Kernel32 directly from pure Java -- no DLL, no native compilation, no cross-compiler toolchain.

## Do Not Call System.console() on Windows

Calling `System.console()` on Windows initializes the JDK's internal JLine terminal (`jdk.internal.le`), which starts its own input pump thread on the same console input handle that Æsh Readline reads. Two readers on one Windows console input queue split the event stream, causing intermittent lost keystrokes — especially when keys overlap or arrive in quick succession.

Æsh Readline itself never calls `System.console()` on Windows: TTY detection uses `GetConsoleMode` via `WinConsoleNative`, and terminal provider selection relies on that check. **Embedders must follow the same rule** — do not call `System.console()` anywhere in a process that uses Æsh Readline on Windows.

## Runtime Requirements

### Java 22+

Applications running on Java 22+ must enable native access for the FFM API:

```
java --enable-native-access=ALL-UNNAMED -jar myapp.jar
```

Without this flag, Java 22-23 prints a warning and Java 24+ throws an `IllegalCallerException`.

### Java 23+

Starting with Java 23, the JVM also warns when JNI loads native libraries without `--enable-native-access`. This means the flag is required on Java 23+ regardless of which implementation is active. The FFM path is the better choice here since it eliminates the DLL entirely.

### Java 8 -- 22

No special flags are needed. The JNI implementation loads `aesh-console.dll` from the JAR automatically.

## How It Works

The multi-release JAR contains two versions of `WinConsoleNative`:

```
terminal-tty.jar
├── org/aesh/terminal/tty/impl/WinConsoleNative.class          (JNI, Java 8)
├── META-INF/versions/22/org/aesh/terminal/tty/impl/WinConsoleNative.class  (FFM, Java 22+)
├── META-INF/MANIFEST.MF                                        (Multi-Release: true)
└── native/windows-x86_64/aesh-console.dll                      (for JNI fallback)
```

The JVM automatically selects the correct class based on the runtime version. No configuration or code changes are needed -- callers like `WinSysTerminal` and `AbstractWindowsTerminal` use the same API regardless of which implementation is active.

Both implementations wrap these Windows Console API functions:

| Function | Purpose |
|----------|---------|
| `GetStdHandle` | Obtain stdin/stdout/stderr handles |
| `GetConsoleMode` / `SetConsoleMode` | Control raw mode, echo, VT processing, mouse input |
| `GetConsoleOutputCP` | Detect console encoding |
| `GetConsoleScreenBufferInfo` | Query terminal width and height |
| `ReadConsoleInputW` | Read key events, mouse events, and window resize events |
| `WriteConsoleW` | Write Unicode output to the console |
| `WaitForSingleObject` | Poll the input pump with timeout for clean shutdown |
| `GetNumberOfConsoleInputEvents` | Drain pending events in batches |

## Building from Source

The multi-release JAR is built automatically based on the JDK used:

**With Java 22+** -- produces a multi-release JAR with both JNI and FFM variants:

```bash
export JAVA_HOME=/path/to/jdk-24
mvn clean package
```

**With Java 8-21** -- produces a standard JAR with only the JNI variant:

```bash
export JAVA_HOME=/path/to/jdk-21
mvn clean package
```

For releases, build with Java 22+ to include the FFM implementation.

## GraalVM Native Image

The FFM implementation is compatible with GraalVM native-image (25+). For GraalVM 23-24, use the JNI implementation -- the `resource-config.json` in the JAR ensures the DLL is included in native images.

## Maven Configuration for Downstream Projects

If your project uses `maven-surefire-plugin` or `maven-exec-plugin` and runs on Java 22+, add the native access flag:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>--enable-native-access=ALL-UNNAMED</argLine>
    </configuration>
</plugin>
```

To avoid breaking builds on older JDKs, use a profile:

```xml
<profile>
    <id>java22-native-access</id>
    <activation>
        <jdk>[22,)</jdk>
    </activation>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <argLine>--enable-native-access=ALL-UNNAMED</argLine>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

## Console Mode Management

When entering raw mode, Æsh Readline builds the console input mode **from scratch** starting at `ENABLE_WINDOW_INPUT` and adding only the needed flags (`ECHO`, `LINE`, `PROCESSED` input as the attributes require, plus mouse/extended flags when mouse tracking is on). It deliberately does not preserve stale flags from the OS default — most importantly `ENABLE_QUICK_EDIT_MODE`, which blocks `ReadConsoleInputW` while text selection is active and would otherwise swallow keystrokes.

The original input *and* output modes are saved at terminal construction and restored on `close()`, so the console is left exactly as it was found.

## CRLF Input Handling

Cooked-mode line discipline (MSYS2/Cygwin pipes and consoles, pasted CRLF text) delivers CR LF per ENTER keypress. `EventDecoder` collapses an LF immediately following a CR into a single submit, so one ENTER yields one line — including when the pair is split across read chunks. Lone CR, lone LF, and repeated sequences pass through unchanged, so explicit blank lines still submit.

## Cygwin and MSYS2

When running under Cygwin or MSYS2, Æsh Readline detects the POSIX-compatible environment and uses PTY-based terminal access instead of the Windows Console API. Neither JNI nor FFM is used in this case.

Detection checks `MSYSTEM` (MSYS2/Git-Bash), the `CYGWIN` variable, `TERM_PROGRAM=mintty`, and a `PWD` starting with `/` — a single heuristic fails when these environments are launched from `cmd.exe`, IDEs, or CI where variables differ.

Terminal routing by environment:

| Environment | Provider | Input Path |
|-------------|----------|------------|
| Native console (conhost, PowerShell, Windows Terminal) | `WinSysTerminal` | `ReadConsoleInputW` console API |
| MSYS2/Cygwin with a real console (ConPTY) | `CygwinPty` + direct console mode | Win32 console mode calls, `stty.exe` as fallback |
| MSYS2/Cygwin with pipes (mintty for native processes) | `CygwinPty` + `stty.exe` | `FileInputStream` on stdin |

`WinSysTerminal` (priority 100) and the Cygwin provider (priority 75) are mutually exclusive via the detection above, so exactly one claims the console. `WinSysTerminal` never enables `ENABLE_VIRTUAL_TERMINAL_INPUT` — it caused duplicate key events — and instead translates Windows virtual key codes to ANSI escape sequences in Java.

## Console Encoding

Console output avoids codepage issues by construction: whenever the output handle is a real console, everything is written via `WriteConsoleW` in UTF-16, which no console codepage setting can garble — regardless of whether VT interpretation is on. This holds for both the JNI (`aesh-console.dll`) and FFM implementations.

Only two paths still depend on charset agreement, and both are safe by construction:

| Path | Encoding | Correct when |
|------|----------|--------------|
| Console output (valid handle) | UTF-16 via `WriteConsoleW` | Always |
| Piped output (no console) | JVM default charset bytes | Always — the pipe reader, not a console codepage, interprets them |
| Console input (`ReadConsoleInputW`) | UTF-16 → JVM charset round trip | Always — both sides use the JVM charset |

Deliberately *not* done: forcing the console output codepage to UTF-8 (`SetConsoleOutputCP(65001)`). That would mutate shared global console state visible to the parent shell and concurrent processes. The UTF-16 output path makes it unnecessary.
