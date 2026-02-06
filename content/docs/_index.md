---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Documentation'
cascade:
  type: docs
---

Welcome to the Æsh project documentation. Build powerful command-line applications in Java with minimal effort.

## Choose Your Library

The Æsh project provides two libraries for different use cases:

### [Æsh](/docs/aesh) - Command Framework (High-Level)

**Build CLI applications with commands, options, and arguments**

Æsh provides a high-level command framework with annotation-based definitions. Perfect for most CLI applications.

```java
@CommandDefinition(name = "deploy", description = "Deploy application")
public class DeployCommand implements Command<CommandInvocation> {
    @Option(shortName = 'e')
    private String environment = "production";
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Deploying to " + environment);
        return CommandResult.SUCCESS;
    }
}
```

**Use Æsh when:**
- Building CLI tools with commands and options
- You want automatic parsing and validation
- You need generated help text and tab completion
- You prefer annotation-based development

**→ [Start with Æsh](/docs/aesh)** (Recommended for most users)

---

### [Æsh Readline](/docs/readline) - Terminal API (Low-Level)

**Direct terminal control for custom applications**

Æsh Readline provides bare-bones terminal input/output APIs. For advanced users who need fine-grained control.

```java
Readline readline = ReadlineBuilder.builder().build();
readline.readline(connection, "prompt> ", input -> {
    // You handle parsing and logic
    connection.write("You entered: " + input + "\n");
});
```

**Use Readline when:**
- Building custom terminal UIs or text-based games
- You need low-level terminal event handling
- Standard command patterns don't fit your use case
- You want maximum flexibility and control

**→ [Explore Readline](/docs/readline)** (For advanced use cases)

---

## Quick Start by Use Case

### I'm building a CLI tool
→ **[Æsh Getting Started](/docs/aesh/getting-started)** - Create commands with options and arguments

### I need an interactive shell
→ **[Æsh Console Runner](/docs/aesh/runners#aeshconsolerunner)** - Interactive shell with history and completion

### I'm making a one-shot CLI utility
→ **[Æsh Runtime Runner](/docs/aesh/runners#aeshruntimerunner)** - Single command execution from command line

### I want to build a text-based game
→ **[Readline Terminal Control](/docs/readline/terminal)** - Low-level terminal manipulation

### I need remote terminal access (SSH/Telnet)
→ **[Readline Connectivity](/docs/readline/connectivity)** - Remote terminal servers

### I want to see working examples
→ **[Æsh Examples](/docs/aesh/examples)** | **[Readline Examples](/docs/readline/examples)**

## Architecture Overview

```
┌──────────────────────────────────────┐
│     Your CLI Application             │
└──────────┬───────────────────────────┘
           │
           ├─── Uses Æsh ───────────────────┐
           │    (High-level commands)       │
           │    • @CommandDefinition        │
           │    • Automatic parsing         │
           │    • Built-in help             │
           │    • Tab completion            │
           │                                │
           └─── Or Uses Readline ───────────┤
                (Low-level terminal)        │
                • readline()                │
                • Manual parsing            │
                • Terminal control          │
                • Event callbacks           │
                                            │
        ┌───────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│      Æsh Readline Core               │
│   (Terminal input/output layer)     │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│   Terminal / Connection              │
│   (Local, SSH, Telnet, WebSocket)    │
└──────────────────────────────────────┘
```

**Note:** Æsh is built on top of Readline. You can use either:
- **Æsh** for the full command framework (most users)
- **Readline** directly for custom terminal applications (advanced users)

## Documentation Structure

### Æsh Documentation

**Getting Started**
- [Installation](/docs/aesh/installation) - Add Æsh to your project
- [Getting Started](/docs/aesh/getting-started) - Your first command
- [Examples & Tutorials](/docs/aesh/examples) - Complete working examples

**Core Concepts**
- [Command Definition](/docs/aesh/command-definition) - Defining commands
- [Options](/docs/aesh/options) - Command-line options
- [Arguments](/docs/aesh/arguments) - Positional arguments
- [Group Commands](/docs/aesh/group-commands) - Command hierarchies
- [Console & Runtime Runners](/docs/aesh/runners) - Execution modes

**Advanced Features**
- [Completers](/docs/aesh/completers) - Tab completion
- [Validators](/docs/aesh/validators) - Input validation
- [Converters](/docs/aesh/converters) - Type conversion
- [Activators](/docs/aesh/activators) - Conditional options
- [Renderers](/docs/aesh/renderers) - Custom output formatting
- [Extensions Library](/docs/aesh/extensions) - Ready-made commands (ls, cd, cat, etc.)

---

### Readline Documentation

**Getting Started**
- [Installation](/docs/readline/installation) - Add Readline to your project
- [Getting Started](/docs/readline/getting-started) - Basic readline usage
- [Examples & Tutorials](/docs/readline/examples) - Complete working examples

**Core API**
- [Readline API](/docs/readline/readline-api) - Core readline functionality
- [Terminal](/docs/readline/terminal) - Terminal control
- [Connection](/docs/readline/connection) - Connection handling
- [History](/docs/readline/history) - Command history

**Advanced Features**
- [Completion](/docs/readline/completion) - Tab completion system
- [Edit Modes](/docs/readline/edit-modes) - Emacs and Vi modes
- [Key Bindings](/docs/readline/key-bindings) - Keyboard shortcuts
- [Remote Connectivity](/docs/readline/connectivity) - SSH, Telnet, WebSocket

## Resources

### Examples Repository
All examples are available at [github.com/aeshell/aesh-examples](https://github.com/aeshell/aesh-examples)
- Æsh console and runtime examples
- Readline terminal examples
- Remote connectivity examples
- Full build instructions

### Community
- **GitHub:** [github.com/aeshell/aesh](https://github.com/aeshell/aesh)
- **Issues:** [Report bugs or request features](https://github.com/aeshell/aesh/issues)
- **Discussions:** [Ask questions and share ideas](https://github.com/aeshell/aesh/discussions)

## License

Æsh and Æsh Readline are licensed under the Apache License 2.0.

