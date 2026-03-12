---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Æsh'
---

Æsh is a Java library for building powerful command-line interfaces with minimal effort. Define commands using annotations, and Æsh handles all the parsing, validation, and execution.

## What is Æsh?

Æsh (Another Extendable SHell) provides a **high-level command framework** that makes it easy to create professional CLI applications. Whether you're building a simple utility or a full-featured shell, Æsh gives you the tools to focus on your commands' logic while it handles the boilerplate.

### The Æsh Approach

```java
@CommandDefinition(name = "deploy", description = "Deploy an application")
public class DeployCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'e', description = "Target environment")
    private String environment = "production";
    
    @Option(shortName = 'v', description = "Enable verbose output")
    private boolean verbose;
    
    @Argument(description = "Application name")
    private String application;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Deploying " + application + " to " + environment);
        return CommandResult.SUCCESS;
    }
}
```

That's it! Æsh automatically:
- Parses `deploy --environment staging myapp`
- Validates required arguments
- Injects values into your fields
- Generates help text
- Handles errors and edge cases

## Key Features

### Command Definition
- **Annotation-based** - Use `@CommandDefinition`, `@Option`, `@Argument` for declarative commands
- **Builder API** - Programmatic command creation if you prefer code over annotations
- **Automatic parsing** - No manual string splitting or argument extraction
- **Type conversion** - Automatic conversion to String, Integer, Boolean, File, and custom types

### Options & Arguments
- **Multiple option types** - Single values, lists, and grouped options
- **Short and long names** - Support both `-v` and `--verbose`
- **Default values** - Sensible defaults with optional overrides
- **Required vs optional** - Mark what's mandatory for your commands
- **Multiple arguments** - Handle variable numbers of positional arguments

### Command Hierarchies
- **Group commands** - Create command trees like `git remote add` or `docker container start`
- **Subcommands** - Unlimited nesting depth for complex CLIs
- **Sub-command mode** - Interactive contexts where subcommands share parent options
- **Inherited options** - Parent options automatically available to subcommands
- **Dynamic registration** - Add and remove commands at runtime

### User Experience
- **Tab completion** - Built-in completers for files, booleans, and enums
- **Custom completers** - Add domain-specific completions
- **[Ghost text suggestions](ghost-text-suggestions)** - Inline suggestions for commands, subcommands, and options as you type
- **Auto-generated help** - `--help` automatically generated from your metadata
- **Input validation** - Built-in and custom validators for options and arguments
- **Error messages** - Clear, helpful error messages for users

### Extensibility
- **Validators** - Ensure option values meet your requirements
- **Converters** - Convert string input to custom types
- **Completers** - Provide tab completion suggestions
- **Suggestion providers** - Inline ghost text suggestions
- **Activators** - Conditionally enable/disable options
- **Renderers** - Customize help output format
- **Custom parsers** - Override default parsing behavior

### Performance & Native Images
- **[Compile-time annotation processor](annotation-processor)** - Optional `aesh-processor` module generates command metadata at build time, eliminating runtime reflection
- **GraalVM native-image friendly** - Generated metadata uses direct `new` calls instead of reflection
- **Dual-mode** - Falls back to reflection automatically if the processor is not used

### Execution Modes
- **Console mode** - Interactive shell with command history and editing (via `AeshConsoleRunner`)
- **Runtime mode** - Single command execution for CLI tools (via `AeshRuntimeRunner`)
- **Programmatic** - Call commands directly from Java code

## Architecture

Æsh is built on top of [Æsh Readline](/docs/readline), which handles the low-level terminal interaction:

```
┌─────────────────────────────────────┐
│      Your Command Classes           │
│   (@CommandDefinition, @Option)     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│          Æsh Framework              │
│  • Command parsing & injection      │
│  • Validation & conversion          │
│  • Help generation                  │
│  • Command registry                 │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        Æsh Readline                 │
│  • Terminal input/output            │
│  • Line editing & history           │
│  • Tab completion                   │
└─────────────────────────────────────┘
```

## Who Should Use Æsh?

**Perfect for:**
- CLI tools and utilities
- Interactive shells and REPLs
- Developer tools and build systems
- System administration utilities
- Command-line applications with complex options

**You'll love Æsh if you want:**
- Quick command development with annotations
- Automatic argument parsing and validation
- Professional help text without manual formatting
- Tab completion out of the box
- Minimal boilerplate code

## Quick Example

Building a simple file search tool:

```java
@CommandDefinition(name = "find", description = "Search for files")
public class FindCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'n', description = "File name pattern")
    private String name;
    
    @Option(shortName = 't', description = "File type", 
            completer = FileTypeCompleter.class)
    private String type;
    
    @Argument(description = "Directory to search")
    private File directory = new File(".");
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Your search logic here
        invocation.println("Searching " + directory + " for " + name);
        return CommandResult.SUCCESS;
    }
}

// Start an interactive shell
public class Main {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(FindCommand.class)
                .prompt("search> ")
                .start();
    }
}
```

Users get automatic help, tab completion, and error handling:
```bash
search> find --help
Usage: find [OPTIONS] [directory]
Search for files

Options:
  -n, --name <name>      File name pattern
  -t, --type <type>      File type
  -h, --help             Display this help message

search> find --name "*.java" /src
Searching /src for *.java
```

## Getting Started

Ready to build your CLI application?

1. **[Installation](installation)** - Add Æsh to your project
2. **[Getting Started](getting-started)** - Create your first command
3. **[Command Definition](command-definition)** - Learn about command structure
4. **[Extensions Library](extensions)** - Use ready-made commands (ls, cd, cat, etc.)
5. **[Examples](examples)** - Explore complete working examples

## Æsh vs Æsh Readline

Not sure which library you need?

- **Use Æsh** (this library) if you're building command-line applications with commands, options, and arguments
- **Use [Æsh Readline](/docs/readline)** if you need low-level terminal control for custom text-based UIs

Æsh is built on Readline and provides the higher-level command abstraction most developers need.

