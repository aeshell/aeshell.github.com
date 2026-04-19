---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Command Registry'
weight: 16
---

The command registry is responsible for storing, managing, and retrieving commands in an Æsh application. It allows dynamic command registration and provides lookup functionality for command execution.

## Overview

Æsh provides two main registry interfaces:

- **`CommandRegistry`** - Read-only registry for command lookup
- **`MutableCommandRegistry`** - Extends `CommandRegistry` with add/remove capabilities

Most applications use `MutableCommandRegistry` directly or let the runner builders manage the registry automatically.

## CommandRegistry Interface

The base interface for command lookup:

```java
public interface CommandRegistry<C extends CommandInvocation> {
    
    // Find a command by name
    CommandContainer<C> getCommand(String name, String line);
    
    // Get all registered command names
    Set<String> getAllCommandNames();
    
    // Get all commands
    Set<CommandContainer<C>> getAllCommands();
    
    // Check if a command exists
    boolean containsCommand(String name);
    
    // Get command aliases
    Set<String> getAliases();
}
```

### Method Details

#### getCommand(String name, String line)

Retrieves a command container by name. The `line` parameter provides the full command line for context (useful for dynamic commands).

```java
CommandContainer<CommandInvocation> cmd = registry.getCommand("copy", "copy --recursive src dest");
```

#### getAllCommandNames()

Returns the names of all registered commands.

```java
Set<String> names = registry.getAllCommandNames();
// e.g., {"copy", "delete", "list", "exit"}
```

#### getAllCommands()

Returns all registered command containers.

```java
Set<CommandContainer<CommandInvocation>> commands = registry.getAllCommands();
for (CommandContainer<CommandInvocation> cmd : commands) {
    System.out.println("Command: " + cmd.getParser().getProcessedCommand().name());
}
```

#### getSubcommandNames(String parentCommandName)

Returns the names of all subcommands registered under a group command. Returns an empty set if the command is not a group or not found.

```java
Set<String> subcommands = registry.getSubcommandNames("git");
// e.g., {"commit", "push", "pull"}
```

This is useful for applications that need to distinguish subcommand names from positional arguments, for example to implement a default subcommand:

```java
Set<String> subcommands = registry.getSubcommandNames("jbang");

public String[] handleArgs(String[] args) {
    if (args.length > 0 && !subcommands.contains(args[0])) {
        // First arg is not a subcommand — prepend default "run"
        return prependRun(args);
    }
    return args;
}
```

#### containsCommand(String name)

Checks if a command with the given name exists.

```java
if (registry.containsCommand("copy")) {
    // Command exists
}
```

## MutableCommandRegistry

Extends `CommandRegistry` with the ability to add and remove commands dynamically.

```java
public interface MutableCommandRegistry<C extends CommandInvocation> extends CommandRegistry<C> {
    
    // Add a command class
    void addCommand(Class<? extends Command<C>> command);
    
    // Add a command instance
    void addCommand(Command<C> command);
    
    // Add a command container
    void addCommand(CommandContainer<C> container);
    
    // Remove a command by name
    void removeCommand(String name);
    
    // Add all commands from another registry
    void addAllCommands(Collection<? extends Command<C>> commands);
    
    // Add all command classes from a collection
    void addAllCommandContainers(Collection<CommandContainer<C>> containers);
}
```

### Creating a Registry

```java
import org.aesh.command.registry.MutableCommandRegistry;
import org.aesh.command.registry.MutableCommandRegistryBuilder;

// Create an empty registry
MutableCommandRegistry<CommandInvocation> registry = MutableCommandRegistryBuilder.builder().build();

// Create a registry with initial commands
MutableCommandRegistry<CommandInvocation> registry = MutableCommandRegistryBuilder.builder()
        .command(CopyCommand.class)
        .command(DeleteCommand.class)
        .command(ListCommand.class)
        .build();
```

### Adding Commands

#### By Class

```java
registry.addCommand(CopyCommand.class);
registry.addCommand(DeleteCommand.class);
```

#### By Instance

```java
registry.addCommand(new CopyCommand());
registry.addCommand(new ConfiguredCommand(config));
```

#### Multiple Commands

```java
List<Command<CommandInvocation>> commands = Arrays.asList(
        new CopyCommand(),
        new DeleteCommand(),
        new ListCommand()
);
registry.addAllCommands(commands);
```

### Removing Commands

```java
registry.removeCommand("copy");
```

### Using with Runners

```java
MutableCommandRegistry<CommandInvocation> registry = MutableCommandRegistryBuilder.builder()
        .command(Command1.class)
        .command(Command2.class)
        .build();

AeshConsoleRunner.builder()
        .commandRegistry(registry)
        .addExitCommand()
        .start();
```

## Dynamic Command Registration

One of the powerful features of Æsh is the ability to add or remove commands at runtime.

### Example: Plugin System

```java
@CommandDefinition(name = "plugin", description = "Manage plugins")
public class PluginCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'a', description = "Action: load, unload, list")
    private String action;
    
    @Argument(description = "Plugin name")
    private String pluginName;
    
    private final MutableCommandRegistry<CommandInvocation> registry;
    
    public PluginCommand(MutableCommandRegistry<CommandInvocation> registry) {
        this.registry = registry;
    }
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        switch (action) {
            case "load":
                loadPlugin(pluginName, invocation);
                break;
            case "unload":
                unloadPlugin(pluginName, invocation);
                break;
            case "list":
                listPlugins(invocation);
                break;
        }
        return CommandResult.SUCCESS;
    }
    
    private void loadPlugin(String name, CommandInvocation invocation) {
        try {
            // Load plugin class dynamically
            Class<?> pluginClass = Class.forName("com.example.plugins." + name + "Command");
            @SuppressWarnings("unchecked")
            Class<? extends Command<CommandInvocation>> cmdClass = 
                    (Class<? extends Command<CommandInvocation>>) pluginClass;
            
            registry.addCommand(cmdClass);
            invocation.println("Plugin '" + name + "' loaded successfully.");
        } catch (ClassNotFoundException e) {
            invocation.println("Plugin not found: " + name);
        }
    }
    
    private void unloadPlugin(String name, CommandInvocation invocation) {
        if (registry.containsCommand(name)) {
            registry.removeCommand(name);
            invocation.println("Plugin '" + name + "' unloaded.");
        } else {
            invocation.println("Plugin not loaded: " + name);
        }
    }
    
    private void listPlugins(CommandInvocation invocation) {
        invocation.println("Loaded commands:");
        for (String name : registry.getAllCommandNames()) {
            invocation.println("  - " + name);
        }
    }
}
```

### Example: Context-Aware Commands

```java
public class ContextAwareShell {
    
    private final MutableCommandRegistry<CommandInvocation> registry;
    private boolean adminMode = false;
    
    public ContextAwareShell() {
        this.registry = MutableCommandRegistryBuilder.builder()
                .command(HelpCommand.class)
                .command(LoginCommand.class)
                .build();
    }
    
    public void enableAdminMode() {
        if (!adminMode) {
            registry.addCommand(DeleteAllCommand.class);
            registry.addCommand(ConfigCommand.class);
            registry.addCommand(ShutdownCommand.class);
            adminMode = true;
        }
    }
    
    public void disableAdminMode() {
        if (adminMode) {
            registry.removeCommand("deleteall");
            registry.removeCommand("config");
            registry.removeCommand("shutdown");
            adminMode = false;
        }
    }
    
    public void start() {
        AeshConsoleRunner.builder()
                .commandRegistry(registry)
                .addExitCommand()
                .prompt(this::getPrompt)
                .start();
    }
    
    private String getPrompt() {
        return adminMode ? "[admin]# " : "[user]$ ";
    }
}
```

## CommandContainer

The `CommandContainer` wraps a command and its parser. It's used internally by the registry.

```java
public interface CommandContainer<C extends CommandInvocation> {
    
    // Get the command parser
    CommandLineParser<C> getParser();
    
    // Check if this is a group command
    boolean isGroupCommand();
    
    // Get help information
    String getHelpInfo();
}
```

### Creating Command Containers

For most use cases, the registry handles container creation automatically. For advanced scenarios:

```java
import org.aesh.command.container.CommandContainerBuilder;

CommandContainer<CommandInvocation> container = CommandContainerBuilder.builder()
        .command(MyCommand.class)
        .create();

registry.addCommand(container);
```

## AeshCommandRegistry

The default implementation of `MutableCommandRegistry`:

```java
import org.aesh.command.registry.AeshCommandRegistry;

// Create empty registry
AeshCommandRegistry<CommandInvocation> registry = new AeshCommandRegistry<>();

// Add commands
registry.addCommand(MyCommand.class);
```

## Complete Example

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.command.*;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;
import org.aesh.command.registry.MutableCommandRegistry;
import org.aesh.command.registry.MutableCommandRegistryBuilder;

public class DynamicShell {
    
    private static MutableCommandRegistry<CommandInvocation> registry;
    
    public static void main(String[] args) {
        registry = MutableCommandRegistryBuilder.<CommandInvocation>builder()
                .command(HelpCommand.class)
                .command(RegisterCommand.class)
                .command(ListCommand.class)
                .build();
        
        AeshConsoleRunner.builder()
                .commandRegistry(registry)
                .addExitCommand()
                .prompt("[dynamic]$ ")
                .start();
    }
    
    @CommandDefinition(name = "help", description = "Show available commands")
    public static class HelpCommand implements Command<CommandInvocation> {
        @Override
        public CommandResult execute(CommandInvocation invocation) {
            invocation.println("Available commands:");
            for (String name : registry.getAllCommandNames()) {
                invocation.println("  " + name);
            }
            return CommandResult.SUCCESS;
        }
    }
    
    @CommandDefinition(name = "list", description = "List registered commands")
    public static class ListCommand implements Command<CommandInvocation> {
        @Override
        public CommandResult execute(CommandInvocation invocation) {
            invocation.println("Registered commands: " + registry.getAllCommandNames().size());
            return CommandResult.SUCCESS;
        }
    }
    
    @CommandDefinition(name = "register", description = "Register a new echo command")
    public static class RegisterCommand implements Command<CommandInvocation> {
        
        @Option(shortName = 'n', required = true, description = "Command name")
        private String name;
        
        @Option(shortName = 'm', required = true, description = "Message to echo")
        private String message;
        
        @Override
        public CommandResult execute(CommandInvocation invocation) {
            // Create a simple echo command dynamically
            // Note: In real applications, you'd use more sophisticated techniques
            if (registry.containsCommand(name)) {
                invocation.println("Command '" + name + "' already exists.");
                return CommandResult.FAILURE;
            }
            
            // For demonstration - in practice you'd create actual command classes
            invocation.println("Registered command '" + name + "' (demo only)");
            return CommandResult.SUCCESS;
        }
    }
}
```

## Thread Safety

The default `AeshCommandRegistry` is **not thread-safe**. If you need to modify the registry from multiple threads:

```java
public class ThreadSafeRegistry {
    private final MutableCommandRegistry<CommandInvocation> registry;
    private final Object lock = new Object();
    
    public void addCommand(Class<? extends Command<CommandInvocation>> cmd) {
        synchronized (lock) {
            registry.addCommand(cmd);
        }
    }
    
    public void removeCommand(String name) {
        synchronized (lock) {
            registry.removeCommand(name);
        }
    }
}
```

## Best Practices

1. **Initialize commands before starting** - Register all static commands before calling `start()` to ensure they're available immediately.

2. **Use command classes for reusable commands** - Pass command classes rather than instances when possible for better memory management.

3. **Handle registration errors** - Wrap command registration in try-catch to handle invalid command classes gracefully.

4. **Clean up on shutdown** - If your application registers temporary commands, clean them up before shutdown.

5. **Consider command dependencies** - When removing commands, ensure no other commands depend on them.

```java
// Good: Check before removing
if (registry.containsCommand("helper") && !isDependency("helper")) {
    registry.removeCommand("helper");
}

// Bad: Remove without checking dependencies
registry.removeCommand("helper");
```
