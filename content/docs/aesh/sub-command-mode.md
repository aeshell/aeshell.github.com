---
date: '2026-02-03T10:00:00+01:00'
draft: false
title: 'Sub-Command Mode'
weight: 8
---

Sub-command mode provides an interactive context for group commands where users can execute multiple subcommands without repeating the parent command. This creates a more efficient workflow similar to shells like `nix-shell` or `kubectl exec -it`.

## Overview

Instead of typing the full command path repeatedly:

```
[myapp]$ project build --target=jar
[myapp]$ project test --coverage
[myapp]$ project deploy --env=staging
```

Users can enter sub-command mode and work within that context:

```
[myapp]$ project --name=myapp --verbose
Entering project mode.
Type 'exit' to return.

project[myapp]> build --target=jar
project[myapp]> test --coverage
project[myapp]> deploy --env=staging
project[myapp]> exit
[myapp]$
```

## Enabling Sub-Command Mode

Sub-command mode is enabled by default. When a group command is executed, it can enter sub-command mode by calling `enterSubCommandMode()`:

```java
@CommandDefinition(
    name = "project",
    description = "Project management",
    groupCommands = {BuildCommand.class, TestCommand.class, DeployCommand.class}
)
public class ProjectCommand implements Command<CommandInvocation> {

    @Option(name = "name", shortName = 'n', required = true, description = "Project name")
    private String projectName;

    @Option(name = "verbose", shortName = 'v', hasValue = false, description = "Verbose output")
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Display current settings
        invocation.println("Project: " + projectName);
        invocation.println("Verbose: " + verbose);

        // Enter sub-command mode - prompt changes to "project[myapp]>"
        if (invocation.enterSubCommandMode(this)) {
            invocation.println("Available subcommands: build, test, deploy");
        }

        return CommandResult.SUCCESS;
    }
}
```

## Exiting Sub-Command Mode

Users can exit sub-command mode in several ways:

- Type `exit` (default exit command)
- Type `..` (alternative exit command)
- Press `Ctrl+C`

All of these are configurable via `SubCommandModeSettings`.

## Accessing Parent Values

Subcommands can access values from parent commands in three ways:

### 1. Using @ParentCommand Annotation

The `@ParentCommand` annotation injects the parent command instance:

```java
@CommandDefinition(name = "build", description = "Build the project")
public class BuildCommand implements Command<CommandInvocation> {

    @ParentCommand
    private ProjectCommand parent;

    @Option(name = "target", defaultValue = {"jar"})
    private String target;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Access parent values directly
        String projectName = parent.getProjectName();
        boolean verbose = parent.isVerbose();

        invocation.println("Building " + projectName + " as " + target);
        if (verbose) {
            invocation.println("[VERBOSE] Compiling sources...");
        }
        return CommandResult.SUCCESS;
    }
}
```

### 2. Using getParentValue()

For programmatic access without a direct dependency on the parent class:

```java
@CommandDefinition(name = "test", description = "Run tests")
public class TestCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Get parent values by field name
        String projectName = invocation.getParentValue("projectName", String.class);
        Boolean verbose = invocation.getParentValue("verbose", Boolean.class, false);

        invocation.println("Testing " + projectName);
        return CommandResult.SUCCESS;
    }
}
```

### 3. Using Inherited Options

Mark parent options with `inherited = true` for automatic propagation to subcommands:

```java
@CommandDefinition(name = "project", ...)
public class ProjectCommand implements Command<CommandInvocation> {

    // This option is automatically available to all subcommands
    @Option(name = "verbose", hasValue = false, inherited = true)
    private boolean verbose;
}

@CommandDefinition(name = "status", description = "Show status")
public class StatusCommand implements Command<CommandInvocation> {

    // This field is auto-populated from parent's inherited option
    @Option(name = "verbose", hasValue = false)
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (verbose) {
            invocation.println("[VERBOSE] Detailed status...");
        }
        return CommandResult.SUCCESS;
    }
}
```

You can also access inherited values programmatically:

```java
Boolean verbose = invocation.getInheritedValue("verbose", Boolean.class);
```

## The `context` Command

When in sub-command mode, the built-in `context` command displays current context information:

```
project[myapp]> context
=== Current Context ===
Path: project
Depth: 1

Context: project
  projectName: myapp
  verbose: true
  config: /etc/app.conf

Inherited values:
  verbose: true

Type 'exit' or '..' to return.
```

## Nested Contexts

Sub-command mode supports nesting. When a subcommand is itself a group command, it can enter another level:

```
[myapp]$ module --name=core
Entering module mode.

module[core]> project --name=myapp
Entering project mode.

module:project[myapp]> build
Building myapp...

module:project[myapp]> exit
module[core]> exit
[myapp]$
```

Access values from any level of the context stack:

```java
// Get value from immediate parent
String projectName = invocation.getParentValue("projectName", String.class);

// Values are searched from immediate parent up to root
String moduleName = invocation.getParentValue("moduleName", String.class);
```

## Configuring Sub-Command Mode

Use `SubCommandModeSettings` to customize behavior:

```java
import org.aesh.command.settings.SubCommandModeSettings;

SubCommandModeSettings subCmdSettings = SubCommandModeSettings.builder()
    .enabled(true)                          // Enable/disable globally
    .exitCommand("exit")                    // Primary exit command
    .alternativeExitCommand("..")           // Alternative exit command
    .contextSeparator(":")                  // Separator for nested contexts
    .showArgumentInPrompt(true)             // Show value in prompt: "project[myapp]>"
    .contextCommand("context")              // Command to show context info
    .enterMessage("Entering {name} mode.")  // Message on entry ({name} is replaced)
    .exitHint("Type '{exit}' to return.")   // Exit hint ({exit}, {alt} are replaced)
    .exitOnCtrlC(true)                      // Ctrl+C exits sub-command mode
    .build();

Settings settings = SettingsBuilder.builder()
    .commandRegistry(registry)
    .subCommandModeSettings(subCmdSettings)
    .build();
```

### Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `enabled` | `boolean` | `true` | Enable/disable sub-command mode globally |
| `exitCommand` | `String` | `"exit"` | Primary command to exit sub-command mode |
| `alternativeExitCommand` | `String` | `".."` | Alternative exit command (set to `null` to disable) |
| `contextSeparator` | `String` | `":"` | Separator for nested context paths in prompt |
| `showArgumentInPrompt` | `boolean` | `true` | Show primary argument in prompt (e.g., `project[myapp]>`) |
| `showContextOnEntry` | `boolean` | `true` | Display context values when entering |
| `contextCommand` | `String` | `"context"` | Command to display context (set to `null` to disable) |
| `enterMessage` | `String` | `"Entering {name} mode."` | Message format when entering (supports `{name}`) |
| `exitMessage` | `String` | `null` | Message format when exiting (supports `{name}`) |
| `exitHint` | `String` | `"Type '{exit}' to return."` | Exit hint (supports `{exit}`, `{alt}`) |
| `exitOnCtrlC` | `boolean` | `true` | Whether Ctrl+C exits sub-command mode |

## Tab Completion

Tab completion works seamlessly in sub-command mode. When you press Tab, only subcommands of the current context are shown:

```
project[myapp]> <Tab>
build    deploy   status   test

project[myapp]> bu<Tab>
project[myapp]> build
```

## Complete Example

```java
import org.aesh.command.*;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;
import org.aesh.command.option.ParentCommand;
import org.aesh.command.settings.*;

public class SubCommandModeExample {

    public static void main(String[] args) {
        // Configure sub-command mode
        SubCommandModeSettings subCmdSettings = SubCommandModeSettings.builder()
            .exitHint("Type 'exit' or '..' to return.")
            .build();

        CommandRegistry registry = AeshCommandRegistryBuilder.builder()
            .command(ProjectCommand.class)
            .create();

        Settings settings = SettingsBuilder.builder()
            .commandRegistry(registry)
            .subCommandModeSettings(subCmdSettings)
            .build();

        new ReadlineConsole(settings).start();
    }

    @CommandDefinition(
        name = "project",
        description = "Project management",
        groupCommands = {BuildCommand.class, TestCommand.class}
    )
    public static class ProjectCommand implements Command<CommandInvocation> {

        @Option(name = "name", required = true)
        private String projectName;

        @Option(name = "verbose", hasValue = false, inherited = true)
        private boolean verbose;

        public String getProjectName() { return projectName; }
        public boolean isVerbose() { return verbose; }

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            invocation.println("Project: " + projectName);
            invocation.enterSubCommandMode(this);
            return CommandResult.SUCCESS;
        }
    }

    @CommandDefinition(name = "build", description = "Build project")
    public static class BuildCommand implements Command<CommandInvocation> {

        @ParentCommand
        private ProjectCommand parent;

        @Option(name = "verbose", hasValue = false)
        private boolean verbose;  // Auto-populated from parent's inherited option

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            invocation.println("Building " + parent.getProjectName());
            if (verbose) {
                invocation.println("[VERBOSE] Compilation started...");
            }
            return CommandResult.SUCCESS;
        }
    }

    @CommandDefinition(name = "test", description = "Run tests")
    public static class TestCommand implements Command<CommandInvocation> {

        @Override
        public CommandResult execute(CommandInvocation invocation) {
            String name = invocation.getParentValue("projectName", String.class);
            Boolean verbose = invocation.getInheritedValue("verbose", Boolean.class, false);

            invocation.println("Testing " + name);
            if (verbose) {
                invocation.println("[VERBOSE] Running all test suites...");
            }
            return CommandResult.SUCCESS;
        }
    }
}
```

## Best Practices

1. **Use inherited options for common flags** - Options like `--verbose`, `--debug`, or `--config` that apply to all subcommands should be marked with `inherited = true`.

2. **Keep context prompts informative** - Include the primary argument in the prompt so users know their current context.

3. **Provide the `context` command** - Users may forget what values they set. The `context` command helps them review.

4. **Document available subcommands** - When entering sub-command mode, print a list of available subcommands.

5. **Support both exit methods** - Keep both `exit` and `..` enabled for user convenience.

6. **Handle Ctrl+C gracefully** - By default, Ctrl+C exits sub-command mode. This is usually the expected behavior.

7. **Use @ParentCommand for type safety** - When you need access to multiple parent values, `@ParentCommand` provides compile-time type checking.

8. **Use getParentValue() for loose coupling** - When subcommands should work with different parent types, use `getParentValue()`.
