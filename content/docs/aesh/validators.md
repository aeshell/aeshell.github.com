---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Validators'
weight: 9
---

Validators check option/argument values before the command executes. Validation errors abort execution and display a message to the user.

There are two kinds: **OptionValidator** checks a single option's value, and **CommandValidator** checks the command as a whole (useful for cross-option constraints).

## OptionValidator

Validates individual options or arguments after conversion:

```java
public interface OptionValidator<T extends ValidatorInvocation> {
    void validate(T validatorInvocation) throws OptionValidatorException;
}
```

The `ValidatorInvocation` provides:
- `getValue()` -- the converted value (already the target type, not a String)
- `getCommand()` -- the command instance being populated
- `getAeshContext()` -- the current Æsh context

### Example: Port Range Validator

```java
import org.aesh.command.validator.OptionValidator;
import org.aesh.command.validator.OptionValidatorException;
import org.aesh.command.validator.ValidatorInvocation;

public class PortValidator implements OptionValidator<ValidatorInvocation> {

    @Override
    public void validate(ValidatorInvocation invocation) throws OptionValidatorException {
        int port = (int) invocation.getValue();
        if (port < 1 || port > 65535) {
            throw new OptionValidatorException(
                "Port must be between 1 and 65535, got: " + port);
        }
        if (port < 1024) {
            throw new OptionValidatorException(
                "Ports below 1024 require root privileges. Use a port >= 1024.");
        }
    }
}
```

Use it with `@Option`:

```java
@CommandDefinition(name = "server", description = "Start the server")
public class ServerCommand implements Command<CommandInvocation> {

    @Option(
        name = "port",
        validator = PortValidator.class,
        description = "Server port (1024-65535)"
    )
    private int port = 8080;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Starting server on port " + port);
        return CommandResult.SUCCESS;
    }
}
```

```
$ server --port 80
Ports below 1024 require root privileges. Use a port >= 1024.
$ server --port 9090
Starting server on port 9090
```

## CommandValidator

Validates the entire command after all options are parsed. This is where you check constraints between options:

```java
public interface CommandValidator<C extends Command<CI>, CI extends CommandInvocation> {
    void validate(C command) throws CommandValidatorException;
}
```

### Example: Cross-Option Validation

```java
import org.aesh.command.validator.CommandValidator;
import org.aesh.command.validator.CommandValidatorException;
import org.aesh.command.invocation.CommandInvocation;

public class ServerCommandValidator
        implements CommandValidator<ServerCommand, CommandInvocation> {

    @Override
    public void validate(ServerCommand command) throws CommandValidatorException {
        if (command.useSSL && command.sslCertPath == null) {
            throw new CommandValidatorException(
                "SSL requires a certificate path. Use --cert <path>.");
        }
        if (command.port == 443 && !command.useSSL) {
            throw new CommandValidatorException(
                "Port 443 is reserved for HTTPS. Add --ssl or use a different port.");
        }
    }
}
```

Reference the validator in `@CommandDefinition`:

```java
@CommandDefinition(
    name = "server",
    description = "Start the server",
    validator = ServerCommandValidator.class
)
public class ServerCommand implements Command<CommandInvocation> {

    @Option(name = "port", description = "Server port")
    int port = 80;

    @Option(name = "ssl", hasValue = false, description = "Enable SSL")
    boolean useSSL = false;

    @Option(name = "cert", description = "SSL certificate path")
    String sslCertPath;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        String protocol = useSSL ? "https" : "http";
        invocation.println("Starting " + protocol + " server on port " + port);
        return CommandResult.SUCCESS;
    }
}
```

```
$ server --ssl
SSL requires a certificate path. Use --cert <path>.
$ server --ssl --cert /etc/ssl/server.pem
Starting https server on port 80
```

## Validation Order

1. Each option's value is **converted** (String to target type)
2. Each option's **OptionValidator** runs (if specified)
3. All options are set on the command object
4. The **CommandValidator** runs (if specified)
5. `execute()` is called

If any step fails, execution stops and the error message is displayed.

## Validator vs Converter

- **Validators** check whether a value is acceptable. They answer: *"Is this value allowed?"*
- **Converters** transform the input string into a typed value. They answer: *"What does this string mean?"*

Use a validator when the type is standard (int, String, File) but only certain values are valid. Use a [Converter](../converters) when you need a custom type.
