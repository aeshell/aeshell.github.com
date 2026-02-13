---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Result Handlers'
weight: 12
---

A `ResultHandler` receives callbacks after command execution completes, whether it succeeded, failed, or threw an exception. This is useful for logging, cleanup, notifications, or any post-execution logic that should run regardless of how the command exits.

## ResultHandler Interface

```java
public interface ResultHandler {

    void onSuccess();

    void onFailure(CommandResult result);

    void onValidationFailure(CommandResult result, Exception exception);

    void onExecutionFailure(CommandResult result, CommandException exception);
}
```

| Method | Called When |
|--------|------------|
| `onSuccess()` | Command returned `CommandResult.SUCCESS` |
| `onFailure(result)` | Command returned `CommandResult.FAILURE` |
| `onValidationFailure(result, exception)` | Option or command validation threw an exception |
| `onExecutionFailure(result, exception)` | Command's `execute()` method threw a `CommandException` |

## Basic Example

```java
public class AuditResultHandler implements ResultHandler {

    @Override
    public void onSuccess() {
        System.out.println("[AUDIT] Command completed successfully");
    }

    @Override
    public void onFailure(CommandResult result) {
        System.out.println("[AUDIT] Command failed");
    }

    @Override
    public void onValidationFailure(CommandResult result, Exception exception) {
        System.out.println("[AUDIT] Validation error: " + exception.getMessage());
    }

    @Override
    public void onExecutionFailure(CommandResult result, CommandException exception) {
        System.out.println("[AUDIT] Execution error: " + exception.getMessage());
    }
}
```

## Attaching to a Command

Specify the handler in the `@CommandDefinition` annotation:

```java
@CommandDefinition(
    name = "deploy",
    description = "Deploy application",
    resultHandler = AuditResultHandler.class
)
public class DeployCommand implements Command<CommandInvocation> {

    @Option(required = true, description = "Target environment")
    private String environment;

    @Override
    public CommandResult execute(CommandInvocation invocation)
            throws CommandException {
        if ("production".equals(environment)) {
            // Simulate a check
            throw new CommandException("Production deploy requires approval");
        }
        invocation.println("Deployed to " + environment);
        return CommandResult.SUCCESS;
    }
}
```

With this configuration:
- `deploy --environment staging` prints "Deployed to staging" and triggers `onSuccess()`
- `deploy --environment production` triggers `onExecutionFailure()` with the exception
- `deploy` (missing required option) triggers `onValidationFailure()`
- A command returning `CommandResult.FAILURE` triggers `onFailure()`

## Use Cases

### Logging

```java
public class LoggingResultHandler implements ResultHandler {

    private static final Logger log = Logger.getLogger("commands");

    @Override
    public void onSuccess() {
        log.info("Command succeeded");
    }

    @Override
    public void onFailure(CommandResult result) {
        log.warning("Command returned failure");
    }

    @Override
    public void onValidationFailure(CommandResult result, Exception exception) {
        log.warning("Validation failed: " + exception.getMessage());
    }

    @Override
    public void onExecutionFailure(CommandResult result, CommandException exception) {
        log.severe("Command threw exception: " + exception.getMessage());
    }
}
```

### Cleanup

```java
public class CleanupResultHandler implements ResultHandler {

    private void cleanup() {
        // Release resources, close connections, delete temp files
    }

    @Override
    public void onSuccess() {
        cleanup();
    }

    @Override
    public void onFailure(CommandResult result) {
        cleanup();
    }

    @Override
    public void onValidationFailure(CommandResult result, Exception exception) {
        // No cleanup needed -- command never ran
    }

    @Override
    public void onExecutionFailure(CommandResult result, CommandException exception) {
        cleanup();
    }
}
```

### Metrics

```java
public class MetricsResultHandler implements ResultHandler {

    @Override
    public void onSuccess() {
        Metrics.counter("commands.success").increment();
    }

    @Override
    public void onFailure(CommandResult result) {
        Metrics.counter("commands.failure").increment();
    }

    @Override
    public void onValidationFailure(CommandResult result, Exception exception) {
        Metrics.counter("commands.validation_error").increment();
    }

    @Override
    public void onExecutionFailure(CommandResult result, CommandException exception) {
        Metrics.counter("commands.execution_error").increment();
    }
}
```

## Execution Order

The result handler runs **after** the command's `execute()` method completes (or after validation fails, before `execute()` is called). The sequence is:

1. Parse and populate options/arguments
2. Run option validators (`OptionValidator`)
3. Run command validator (`CommandValidator`)
4. Run `execute()` method
5. Call the appropriate `ResultHandler` method based on the outcome

If validation fails at step 2 or 3, `onValidationFailure()` is called and `execute()` is never invoked.
