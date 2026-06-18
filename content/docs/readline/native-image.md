---
date: '2026-06-18T12:00:00+01:00'
draft: false
title: 'GraalVM Native Image'
weight: 31
---

Aesh Readline supports GraalVM native-image compilation, enabling fast startup and low memory footprint for CLI applications.

## Quick Start

Build a native image with:

```bash
native-image \
  --no-fallback \
  --enable-native-access=ALL-UNNAMED \
  -cp your-app.jar:readline-3.15.jar:terminal-tty-3.15.jar:terminal-api-3.15.jar:readline-api-3.15.jar:terminal-detect-3.15.jar \
  com.example.MyApp
```

The `--enable-native-access=ALL-UNNAMED` flag is required to suppress warnings from the FFM (Foreign Function & Memory) API on Java 22+.

## How It Works

The `terminal-tty` module includes GraalVM reachability metadata in `META-INF/native-image/` that registers:

- **Reflection config** for `sun.misc.Signal` handling and `java.io.Console.isTerminal()`
- **Resource config** for `META-INF/services/` (ServiceLoader) and native DLLs
- **JNI config** for Windows console native methods
- **Proxy config** for `sun.misc.SignalHandler`

No additional configuration is needed for most use cases.

## Terminal Providers in Native Image

The [Terminal Provider SPI]({{< relref "terminal-provider" >}}) discovers providers at runtime. In native-image mode:

| Platform | Provider Used | Mechanism |
|----------|--------------|-----------|
| Linux/macOS | `ExecPtyTerminalProvider` | Spawns `stty`/`tty` processes |
| Windows | `WinSysTerminalProvider` | JNI via `aesh-console.dll` |
| Cygwin/MSYS2 | `CygwinTerminalProvider` | POSIX PTY via Cygwin |

The FFM-based providers (`FfmTerminalProvider` on POSIX, FFM `WinConsoleNative` on Windows) are not available in native-image because GraalVM does not yet support registering FFM downcall descriptors in reachability metadata. The providers fail gracefully and fall through to the next available provider.

## Maven Profile for Native Tests

To run tests as native-image (validates reachability metadata):

```xml
<profile>
    <id>native</id>
    <activation>
        <property>
            <name>env.GRAALVM_HOME</name>
        </property>
    </activation>
    <dependencies>
        <dependency>
            <groupId>org.junit.vintage</groupId>
            <artifactId>junit-vintage-engine</artifactId>
            <version>5.11.4</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.platform</groupId>
            <artifactId>junit-platform-launcher</artifactId>
            <version>1.11.4</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.graalvm.buildtools</groupId>
                <artifactId>native-maven-plugin</artifactId>
                <version>1.1.2</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <id>test-native</id>
                        <goals><goal>test</goal></goals>
                        <phase>test</phase>
                    </execution>
                </executions>
                <configuration>
                    <buildArgs>
                        <arg>--enable-native-access=ALL-UNNAMED</arg>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

Run native tests with:

```bash
GRAALVM_HOME=$JAVA_HOME mvn clean verify -pl terminal-tty -am
```

## Recommended JVM Flags

When running as a regular JAR (not native-image) on Java 22+, add:

```
--enable-native-access=ALL-UNNAMED
```

This enables FFM-based terminal I/O which avoids spawning external `stty` processes. Without this flag, Java 24+ will block FFM access.

For Maven Surefire:

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>--enable-native-access=ALL-UNNAMED</argLine>
    </configuration>
</plugin>
```

## Compatibility

| GraalVM Version | Status |
|----------------|--------|
| GraalVM CE 23 | Supported |
| GraalVM CE 24 | Supported |
| GraalVM CE 25 | Supported |

Both Linux and Windows are supported. The CI pipeline tests native-image on both platforms.

## Troubleshooting

### ExternalTerminal instead of PosixSysTerminal

If your application gets `ExternalTerminal` (no raw mode, no tab completion), the ServiceLoader provider discovery failed. This can happen if:

- The `terminal-tty` JAR is missing from the classpath
- A custom `native-image.properties` file interferes with the `UseServiceLoaderFeature`

Verify with logging:

```bash
native-image -Djava.util.logging.config.file=logging.properties ...
```

With a `logging.properties` that enables `FINE` on `org.aesh.terminal.tty.TerminalBuilder`.

### FFM Warnings on Java 22+

```
WARNING: java.lang.foreign.Linker::downcallHandle has been called ...
WARNING: Use --enable-native-access=ALL-UNNAMED to avoid a warning
```

Add `--enable-native-access=ALL-UNNAMED` to your JVM or native-image arguments.

### Windows: MissingForeignRegistrationError

If you see this error in native-image on Windows, ensure you're using aesh-readline 3.15+. Older versions used the FFM `WinConsoleNative` which requires downcall registration. Version 3.15+ uses the JNI version automatically.
