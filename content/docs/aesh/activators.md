---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Activators'
weight: 11
---

Activators control whether options, arguments, or entire commands are available/enabled.

## OptionActivator

Enable or disable an option based on conditions:

```java
public class SslOptionActivator implements OptionActivator {

    @Override
    public boolean isActivated(CompleterInvocation invocation) {
        MyCommand cmd = (MyCommand) invocation.getCommand();
        // SSL options only active if secure mode is enabled
        return cmd.secureMode;
    }
}
```

Usage:

```java
@CommandDefinition(name = "server")
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
        return CommandResult.SUCCESS;
    }
}
```

## CommandActivator

Control whether a command is visible/available:

```java
public class AdminCommandActivator implements CommandActivator {

    @Override
    public boolean isActivated(CommandInvocation invocation) {
        // Only show admin commands if user is admin
        return isAdmin(invocation);
    }

    private boolean isAdmin(CommandInvocation invocation) {
        // Check if user has admin privileges
        // This could check environment variables, config files, etc.
        return System.getProperty("user.role", "user").equals("admin");
    }
}
```

Usage:

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

## Conditional Validation

Use activators to conditionally require options:

```java
public class OutputFileActivator implements OptionActivator {

    @Override
    public boolean isActivated(CompleterInvocation invocation) {
        MyCommand cmd = (MyCommand) invocation.getCommand();
        // Output file option only active when not verbose
        return !cmd.verbose;
    }
}

@CommandDefinition(name = "process")
public class ProcessCommand implements Command<CommandInvocation> {

    @Option(name = "verbose", hasValue = false, description = "Verbose output")
    private boolean verbose = false;

    @Option(
        name = "output",
        activator = OutputFileActivator.class,
        required = true,
        description = "Output file (required unless verbose)"
    )
    private String outputFile;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}
```

When `--verbose` is used, the `--output` option is hidden and not required.

## Accessing Runtime Context

Activators can access the Aesh context:

```java
public class ContextAwareActivator implements OptionActivator {

    @Override
    public boolean isActivated(CompleterInvocation invocation) {
        AeshContext context = invocation.getAeshContext();
        // Access runtime context for more complex conditions
        return context.someCondition();
    }
}
```
