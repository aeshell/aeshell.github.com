---
date: '2026-11-03T12:00:00+01:00'
draft: false
title: 'Method Commands'
weight: 4
---

Method commands declare a CLI command as a plain Java method instead of a `Command` class. Annotate a method with `@MethodCommand` and aesh maps its parameters to options using the same machinery as field-based options.

```java
public class GitCommands {

    @MethodCommand(name = "commit", description = "Record changes to the repository")
    public CommandResult commit(
            @Option(name = "message", shortName = 'm', description = "Commit message") String message,
            @Option(name = "all", hasValue = false, description = "Stage all modified files") boolean all,
            CommandInvocation invocation) {
        invocation.println("Committing: " + message);
        return CommandResult.SUCCESS;
    }
}

// Register on a caller-supplied instance:
CommandContainer<CommandInvocation> container =
        new AeshCommandContainerBuilder<CommandInvocation>()
                .create(new GitCommands(), GitCommands.class.getMethod("commit", String.class, boolean.class, CommandInvocation.class));
```

## Registration

The instance model comes first: you supply the object the method runs on.

| Call | Meaning |
|------|---------|
| `create(instance, method)` | Register one annotated method on the given instance |
| `create(owner, commandName)` | Scan the instance for a `@MethodCommand` with a matching name or alias |
| `create(ownerClass, commandName)` | Scan a class for a static `@MethodCommand` with a matching name |

When `name` is empty, the command name is derived verbatim from the method name. Names must be unique within the declaring class — overloaded methods that collide fail registration.

## Method Rules

- Return `void` or `CommandResult`. Anything else fails registration.
- Visibility must be `public`, `protected`, or package-private. `private` and `abstract` methods are rejected.
- Only `CommandInvocation` parameters pass through untouched (they must not carry option annotations). Every other parameter needs an option annotation.

## Parameter Mapping

| Annotation | Rule |
|------------|------|
| `@Option` | Requires an **explicit `name`** — parameter names depend on the `-parameters` compiler flag, so relying on them would make the CLI surface change with build flags |
| `@Argument` / `@Arguments` | Require an **explicit `paramLabel`** for the same reason |
| `@OptionList` | Requires an explicit `name`; `String[]` and `Collection` element types supported |
| `@OptionGroup`, `@Mixin` | Rejected on method parameters |
| `CommandInvocation` | Passed straight through at execution time, no annotation allowed |

`Optional<T>` parameters are supported — the value is unwrapped before conversion. All other option attributes (`required`, `defaultValue`, `converter`, `validator`, `completer`, `allowedValues`, `visibility`, …) work exactly as on fields.

## Annotation Processor

The annotation processor generates a companion class per `@MethodCommand` with direct calls — no `Method.invoke` reflection at runtime. No code changes are needed; the same source works with and without the processor on the classpath.

## When to Use Them

- Grouping related subcommands as methods on one class instead of one class per command.
- Commands that are thin adapters over existing service methods.
- Picocli-style method commands (`@Command` on a method with `names =`).

Use `Command` classes when you need field reuse across executions, `@Mixin` composition, or `@ParentCommand` injection — none of which apply to method parameters.
