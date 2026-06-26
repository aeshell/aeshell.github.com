---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Æsh - Java Libraries for Command-Line Interfaces'
---

Æsh (Another Extendable SHell) is a collection of Java libraries for building command-line interfaces, interactive shells, and terminal applications.

## Overview

The Æsh project provides two complementary libraries:

- **[Æsh](/docs/aesh)** - High-level command framework with annotation-based definitions
- **[Æsh Readline](/docs/readline)** - Low-level terminal input/output and line editing

Both libraries are production-ready, cross-platform (POSIX and Windows), and actively maintained.

## Æsh - Command Framework

Æsh provides a high-level API for building CLI applications with commands, options, and arguments. It handles parsing, validation, type conversion, and help generation automatically.

### Example

```java
@CommandDefinition(name = "deploy", description = "Deploy an application")
public class DeployCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'e', defaultValue = "production", 
            description = "Target environment")
    private String environment;
    
    @Option(shortName = 'f', hasValue = false, 
            description = "Force deployment")
    private boolean force;
    
    @Argument(description = "Application name", required = true)
    private String application;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Deploying " + application + " to " + environment);
        if (force) {
            invocation.println("Force flag enabled");
        }
        return CommandResult.SUCCESS;
    }
}
```

Start an interactive console or execute single commands:

```java
// Interactive console with history and tab completion
AeshConsoleRunner.builder()
    .command(DeployCommand.class)
    .prompt("[deploy-tool]$ ")
    .start();

// Single command execution (for CLI tools)
AeshRuntimeRunner.builder()
    .command(DeployCommand.class)
    .args(args)
    .execute();
```

### Key Capabilities

**Command Definition**
- Annotation-based (`@CommandDefinition`, `@Option`, `@Argument`) or builder API
- Optional [compile-time annotation processor](/docs/aesh/annotation-processor) eliminates runtime reflection
- Automatic parsing and type conversion (String, Integer, Boolean, File, Enum, custom types)
- Support for short (`-v`) and long (`--verbose`) option names
- Required and optional parameters with default values

**Command Organization**
- Command groups and subcommands (e.g., `git remote add`, `docker container start`)
- Unlimited nesting depth for complex hierarchies
- Dynamic command registration and removal at runtime

**User Experience**
- Automatic help generation (`--help`) with [visibility levels](/docs/aesh/options#visibility-levels)
- Tab completion for commands, options, and values
- Shell completion script generation for bash, zsh, and fish
- Built-in completers for files, booleans, enums
- [Ghost text suggestions](/docs/aesh/ghost-text-suggestions) as you type
- Command history and line editing (via Readline)
- Clear error messages and validation feedback

**Extensibility**
- **Validators** - Custom validation rules for options and arguments
- **Converters** - Convert string input to custom types
- **Completers** - Domain-specific tab completion
- **Activators** - Conditionally enable/disable options based on other options
- **Renderers** - Customize help output format

**Execution Modes**
- **Console mode** - Interactive shell with readline features
- **Runtime mode** - Single command execution from command-line arguments
- **Programmatic** - Direct command invocation from Java code

**Performance & Native Images**
- [Annotation processor](/docs/aesh/annotation-processor) shifts all reflection to compile time -- **3-4x faster** startup than the reflection path
- **67-655x faster** command registration than picocli in benchmarks
- **8-21x faster** command-line parsing than picocli
- GraalVM native-image friendly -- generated code uses direct `new` calls, no reflection configuration needed
- Zero external dependencies

Coming from picocli? See the [Migration Guide](/docs/aesh/migrating-from-picocli).

### Use Cases

- CLI tools and utilities
- Interactive shells and REPLs
- Build systems and deployment tools
- System administration utilities
- Developer tools and code generators

## Æsh Readline - Terminal API

Æsh Readline provides low-level terminal input handling with GNU Readline compatibility. It's used by Æsh internally but can be used standalone for custom terminal applications.

### Example

```java
TerminalConnection connection = new TerminalConnection();
Readline readline = ReadlineBuilder.builder()
    .history(new FileHistory(new File(".history"), 100))
    .build();

read(connection, readline);
connection.openBlocking();

private static void read(TerminalConnection connection, Readline readline) {
    readline.readline(connection, "prompt> ", input -> {
        if (input != null && input.equals("exit")) {
            connection.close();
        } else {
            connection.write("Echo: " + input + "\n");
            read(connection, readline);
        }
    });
}
```

### Key Capabilities

**Line Editing**
- Full line editing with undo/redo support
- Emacs and Vi editing modes
- Customizable key bindings
- Copy/paste with paste buffer

**History Management**
- Persistent history with file storage
- History search (Ctrl+R)
- Configurable history size
- Per-session or global history

**Input Features**
- Tab completion with custom completers
- Password masking for sensitive input
- Multi-line input support
- Prompt customization

**Terminal Control**
- Cursor positioning and movement
- Screen clearing and manipulation
- Color and formatting (ANSI escape sequences)
- Terminal size detection

**Remote Connectivity**
- SSH server support (via `terminal-ssh`)
- Telnet server support (via `terminal-telnet`)
- WebSocket server support (via `terminal-http`)
- Unified API for local and remote terminals

**Platform Support**
- POSIX systems (Linux, macOS, BSD)
- Windows (native and WSL)
- Remote terminals via SSH/Telnet/WebSocket

### Use Cases

- Custom terminal applications and text-based UIs
- Text editors and viewers
- Terminal multiplexers
- Interactive shells with custom syntax
- Terminal-based games and animations
- Remote shell applications

## Architecture

```
┌──────────────────────────────────────────────┐
│        Your Application                      │
└──────────────┬───────────────────────────────┘
               │
               ├─── Uses Æsh Framework ──────────┐
               │    • Command definitions        │
               │    • Automatic parsing          │
               │    • Validation & conversion    │
               │    • Help generation            │
               │                                 │
               └─── Or Uses Readline Directly ───┤
                    • Terminal input/output      │
                    • Manual command handling    │
                    • Custom UI control          │
                                                 │
            ┌────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────┐
│         Æsh Readline (Core Library)          │
│  • Terminal abstraction                      │
│  • Line editing & history                    │
│  • Completion & key bindings                 │
│  • Connection management                     │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│    Terminal / Connection Layer               │
│  • Local TTY (POSIX/Windows)                 │
│  • SSH connections                           │
│  • Telnet connections                        │
│  • WebSocket connections                     │
└──────────────────────────────────────────────┘
```

**Note:** Æsh is built on top of Readline. Most users building CLI applications should use Æsh. Use Readline directly only when you need low-level terminal control.

## Installation

### Maven

**Æsh (Command Framework):**
```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh</artifactId>
    <version>3.16</version>
</dependency>
```

**Æsh Readline (Terminal API):**
```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>readline</artifactId>
    <version>3.16</version>
</dependency>
```

### Gradle

**Æsh:**
```groovy
implementation 'org.aesh:aesh:3.16'
```

**Readline:**
```groovy
implementation 'org.aesh:readline:3.16'
```

**Extensions (Optional - Ready-made commands):**
```groovy
implementation 'org.aesh:aesh-extensions:1.7'
```

## Documentation

- **[Getting Started Guide](/docs)** - Choose the right library and get started
- **[Æsh Documentation](/docs/aesh)** - Complete command framework reference
- **[Readline Documentation](/docs/readline)** - Terminal API reference
- **[Extensions Library](/docs/aesh/extensions)** - Pre-built commands (ls, cd, cat, etc.)
- **[Examples Repository](https://github.com/aeshell/aesh-examples)** - Working code examples

## Requirements

- Java 8 or later
- No external dependencies (beyond JDK)

## Project Resources

- **GitHub Repository:** [github.com/aeshell/aesh](https://github.com/aeshell/aesh)
- **Examples Repository:** [github.com/aeshell/aesh-examples](https://github.com/aeshell/aesh-examples)
- **Extensions Repository:** [github.com/aeshell/aesh-extensions](https://github.com/aeshell/aesh-extensions)
- **Issue Tracker:** [github.com/aeshell/aesh/issues](https://github.com/aeshell/aesh/issues)
- **Discussions:** [github.com/aeshell/aesh/discussions](https://github.com/aeshell/aesh/discussions)

## License

Apache License 2.0

## Related Projects

Æsh has been used in several open-source projects:

- **WildFly** - Application server CLI
- **JBoss Tools** - Command-line interfaces
- Various system administration and developer tools

See the [GitHub repository](https://github.com/aeshell/aesh) for more information.
