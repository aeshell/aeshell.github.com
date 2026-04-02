---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Activators'
weight: 11
---

Activators control whether options or entire commands are available. Deactivated options are hidden from tab completion and help, and deactivated commands are invisible to the user.

## OptionActivator

Enable or disable an option based on conditions. The `ParsedCommand` gives you access to the current command and the values of other options that have been parsed so far:

```java
public interface OptionActivator {
    boolean isActivated(ParsedCommand parsedCommand);
}
```

### Example: Conditional Options

SSL-related options should only appear when `--secure` is used:

```java
import org.aesh.command.activator.OptionActivator;
import org.aesh.command.impl.internal.ParsedCommand;

public class SslOptionActivator implements OptionActivator {

    @Override
    public boolean isActivated(ParsedCommand parsedCommand) {
        // SSL options only active if --secure has been specified
        return parsedCommand.findLongOptionNoActivatorCheck("secure")
                .getValue() != null;
    }
}
```

Usage:

```java
@CommandDefinition(name = "server", description = "Start the server")
public class ServerCommand implements Command<CommandInvocation> {

    @Option(name = "secure", hasValue = false, description = "Enable secure mode")
    private boolean secureMode = false;

    @Option(
        name = "cert",
        activator = SslOptionActivator.class,
        description = "SSL certificate (only in secure mode)"
    )
    private String certPath;

    @Option(
        name = "key",
        activator = SslOptionActivator.class,
        description = "SSL private key (only in secure mode)"
    )
    private String keyPath;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (secureMode) {
            invocation.println("Starting HTTPS with cert=" + certPath);
        } else {
            invocation.println("Starting HTTP server");
        }
        return CommandResult.SUCCESS;
    }
}
```

When `--secure` is not set, `--cert` and `--key` are hidden from tab completion and `--help` output.

## CommandActivator

Control whether an entire command is visible and available:

```java
public interface CommandActivator {
    boolean isActivated(ParsedCommand command);
}
```

### Example: Admin-Only Commands

```java
import org.aesh.command.activator.CommandActivator;
import org.aesh.command.impl.internal.ParsedCommand;

public class AdminCommandActivator implements CommandActivator {

    @Override
    public boolean isActivated(ParsedCommand command) {
        return "admin".equals(System.getProperty("user.role", "user"));
    }
}
```

```java
@CommandDefinition(
    name = "shutdown",
    description = "Shutdown the server",
    activator = AdminCommandActivator.class
)
public class ShutdownCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Shutting down...");
        return CommandResult.SUCCESS;
    }
}
```

The `shutdown` command only appears in tab completion and help when `user.role` is `admin`. Users without admin role cannot discover or execute it.

## Conditional Required Options

Combine activators with `required = true` to make options conditionally required:

```java
public class OutputFileActivator implements OptionActivator {

    @Override
    public boolean isActivated(ParsedCommand parsedCommand) {
        // Output file only required when not in verbose mode
        return parsedCommand.findLongOptionNoActivatorCheck("verbose")
                .getValue() == null;
    }
}

@CommandDefinition(name = "process", description = "Process data")
public class ProcessCommand implements Command<CommandInvocation> {

    @Option(name = "verbose", hasValue = false, description = "Verbose output to console")
    private boolean verbose = false;

    @Option(
        name = "output",
        activator = OutputFileActivator.class,
        required = true,
        description = "Output file (required unless --verbose)"
    )
    private String outputFile;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (verbose) {
            invocation.println("Processing... (verbose output)");
        } else {
            invocation.println("Writing output to " + outputFile);
        }
        return CommandResult.SUCCESS;
    }
}
```

When `--verbose` is used, `--output` is deactivated and not required. Without `--verbose`, `--output` is required.
