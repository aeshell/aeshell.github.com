---
date: '2026-02-06T15:00:00+01:00'
draft: false
title: 'Æsh Readline'
---

Æsh Readline is a low-level library for reading terminal input and handling terminal events, with the goal to support most GNU Readline features.

## Overview

Æsh Readline provides a **bare-bones API** for advanced users who need fine-grained control over terminal interaction. Unlike Æsh, which offers a high-level command framework with annotations and automatic parsing, Readline gives you direct access to the terminal layer.

### When to Use Readline

**Use Æsh Readline if you:**
- Need direct control over terminal input/output
- Want to build custom terminal applications from scratch
- Require low-level terminal event handling
- Are building line-based interfaces that don't fit the command pattern
- Need minimal overhead and maximum flexibility

**Use Æsh instead if you:**
- Want to build command-line applications with commands and options
- Prefer annotation-based command definitions (`@CommandDefinition`, `@Option`)
- Need automatic argument parsing and validation
- Want built-in help generation and command registration

Think of it this way: Æsh is built **on top of** Æsh Readline. Æsh provides the easy-to-use command framework, while Readline provides the underlying terminal interaction primitives.

## Key Features

Æsh Readline offers comprehensive terminal input handling:

- **Line Editing** - Undo/redo support with customizable key bindings
- **History Management** - Search, persistence, and navigation
- **Tab Completion** - Extensible completion system
- **Input Masking** - Password and sensitive input handling
- **Paste Buffer** - Copy/paste operations
- **Edit Modes** - Both Emacs and Vi editing modes
- **Cross-Platform** - Works on POSIX systems and Windows
- **History Configuration** - Configurable file storage and buffer size
- **Stream Support** - Standard out and standard error handling
- **Shell Features** - Redirect, alias, and pipeline support
- **Remote Connectivity** - SSH, Telnet, and WebSocket terminal servers
- **[Terminal Environment](terminal-environment)** - Centralized terminal detection (type, color depth, multiplexers)
- **[Color Detection](color-detection)** - Automatic terminal theme detection with batch queries and fallbacks
- **[Terminal Colors](terminal-colors)** - RGB/HSL colors, theme-aware styling, and color depth adaptation
- **[Device Attributes](device-attributes)** - DA1/DA2 terminal capability queries
- **[Terminal Images](terminal-images)** - Sixel, Kitty, and iTerm2 inline image support

## Architecture

```
┌─────────────────────────────────────┐
│           Your Application          │
│    (Terminal UI, Shell, etc.)       │
└──────────────┬──────────────────────┘
               │ Uses
               ▼
┌─────────────────────────────────────┐
│         Æsh Readline API            │
│  ┌───────────────────────────────┐  │
│  │ • readline()                  │  │
│  │ • Prompt handling             │  │
│  │ • Event callbacks             │  │
│  │ • Terminal control            │  │
│  └───────────────────────────────┘  │
└──────────────┬──────────────────────┘
               │ Controls
               ▼
┌─────────────────────────────────────┐
│      Terminal / Connection          │
│  (Local TTY, SSH, Telnet, WebSocket)│
└─────────────────────────────────────┘
```

## What You Get

Æsh Readline provides the building blocks for terminal applications:

- **Input Reading** - Non-blocking, event-driven input reading
- **Terminal Control** - Cursor movement, screen clearing, colors
- **Event Handling** - Callbacks for input, completion, history
- **Connection Abstraction** - Same API for local and remote terminals
- **Key Bindings** - Customizable keyboard shortcuts
- **Edit Operations** - Low-level text editing primitives

## What You Handle

With Readline's bare-bones API, you're responsible for:

- **Input Parsing** - No automatic command parsing or argument extraction
- **Command Dispatch** - You implement the logic to handle user input
- **Error Handling** - No automatic validation or error messages
- **Help System** - No built-in help generation
- **Application Flow** - You control the read loop and application lifecycle

## Quick Comparison

| Feature | Æsh (High-Level) | Æsh Readline (Low-Level) |
|---------|------------------|--------------------------|
| **API Style** | Annotation-based, declarative | Callback-based, imperative |
| **Command Parsing** | Automatic | Manual |
| **Learning Curve** | Gentle | Steeper |
| **Flexibility** | Structured commands | Complete freedom |
| **Use Case** | CLI tools, shells | Custom terminal UIs, games |
| **Setup Complexity** | Minimal | More involved |
| **Control Level** | High-level | Low-level |

## Example: The Difference

### With Æsh (High-Level)
```java
@CommandDefinition(name = "greet", description = "Greet someone")
public class GreetCommand implements Command<CommandInvocation> {
    @Option(shortName = 'n')
    private String name;
    
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name);
        return CommandResult.SUCCESS;
    }
}
// Æsh handles: parsing, validation, help, execution
```

### With Æsh Readline (Low-Level)
```java
Readline readline = ReadlineBuilder.builder().build();
readline.readline(connection, "prompt> ", input -> {
    // You parse the input manually
    if (input.startsWith("greet ")) {
        String name = input.substring(6);
        connection.write("Hello, " + name + "\n");
    }
    // You handle the next read
    readNext(connection, readline);
});
// You handle: parsing, validation, help, flow control
```

## Getting Started

If you're certain Readline's low-level API is what you need:

1. Start with [Installation](installation) to add Readline to your project
2. Follow the [Getting Started Guide](getting-started) for basic usage
3. Explore the [Readline API](readline-api) for core functionality
4. Check out [Examples and Tutorials](examples) for complete working code

## Still Unsure?

If you're building a command-line tool with commands, options, and arguments, **you probably want [Æsh](/docs/aesh)** instead. It will save you significant time and effort.
Remember, all of the features in readline are provided to you in Æsh as well.

Use Readline when you need the raw terminal power for custom applications like:
- Text-based games (see the [Snake example](examples#4-snake-game))
- Custom shells with non-standard syntax
- Terminal multiplexers or viewers
- Applications with complex terminal layouts
- Low-latency terminal applications

