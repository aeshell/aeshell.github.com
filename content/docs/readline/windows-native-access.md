---
date: '2026-04-18T12:00:00+02:00'
draft: false
title: 'Windows Native Access'
weight: 16
---

On Windows, Æsh Readline needs access to the Windows Console API (Kernel32) for raw terminal input, console mode control, and terminal size detection. Starting with version 3.6, the `terminal-tty` module ships as a **multi-release JAR** with two implementations:

| Java Version | Implementation | Native Code Required |
|-------------|----------------|---------------------|
| 8 -- 21 | JNI (`aesh-console.dll`) | Yes |
| 22+ | FFM (`java.lang.foreign`) | No |

On Java 22+, the Foreign Function & Memory API calls Kernel32 directly from pure Java -- no DLL, no native compilation, no cross-compiler toolchain.

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

## Cygwin and MSYS2

When running under Cygwin or MSYS2, Æsh Readline detects the POSIX-compatible environment and uses PTY-based terminal access instead of the Windows Console API. Neither JNI nor FFM is used in this case.
