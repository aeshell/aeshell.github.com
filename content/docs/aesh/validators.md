---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Validators'
weight: 9
---

Validators validate option/argument values before command execution.

## OptionValidator

Validate individual options or arguments:

```java
public class PortValidator implements OptionValidator {

    @Override
    public void validate(String value) throws ValidatorException {
        try {
            int port = Integer.parseInt(value);
            if (port < 1 || port > 65535) {
                throw new ValidatorException("Port must be between 1 and 65535");
            }
        } catch (NumberFormatException e) {
            throw new ValidatorException("Invalid port number: " + value);
        }
    }
}
```

Usage in command:

```java
@CommandDefinition(name = "server")
public class ServerCommand implements Command<CommandInvocation> {

    @Option(
        name = "port", 
        validator = PortValidator.class,
        description = "Server port"
    )
    private int port = 8080;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Starting server on port " + port);
        return CommandResult.SUCCESS;
    }
}
```

## CommandValidator

Validate the entire command before execution:

```java
public class ServerCommandValidator implements CommandValidator<ServerCommand> {

    @Override
    public void validate(ServerCommand command) throws ValidatorException {
        if (command.useSSL && !command.sslCertPath) {
            throw new ValidatorException("SSL requires a certificate path");
        }
        if (command.port == 443 && !command.useSSL) {
            throw new ValidatorException("Port 443 requires SSL");
        }
    }
}
```

Usage:

```java
@CommandDefinition(
    name = "server",
    validator = ServerCommandValidator.class
)
public class ServerCommand implements Command<CommandInvocation> {

    @Option(name = "port", description = "Server port")
    private int port = 80;

    @Option(name = "ssl", hasValue = false, description = "Enable SSL")
    private boolean useSSL = false;

    @Option(name = "cert", description = "SSL certificate path")
    private String sslCertPath;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Port and SSL are validated before this runs
        return CommandResult.SUCCESS;
    }
}
```

## ValidatorException

Throw this exception when validation fails:

```java
throw new ValidatorException("Custom error message");
```

The error message will be displayed to the user, and command execution will be aborted.
