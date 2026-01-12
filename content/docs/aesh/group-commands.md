---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Group Commands'
weight: 7
---

Group commands allow you to create hierarchical command structures, similar to `git` with subcommands like `commit`, `push`, `pull`.

## @GroupCommandDefinition

Use the `@GroupCommandDefinition` annotation to define a command group:

```java
@GroupCommandDefinition(
    name = "git",
    description = "Version control system"
)
public class GitCommand implements GroupCommand<CommandInvocation> {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Git commands: commit, push, pull");
        return CommandResult.SUCCESS;
    }
}
```

## Subcommands

Define subcommands as regular commands with `@CommandDefinition`:

```java
@CommandDefinition(name = "commit", description = "Commit changes")
public class CommitCommand implements Command<CommandInvocation> {

    @Option(shortName = 'm', description = "Commit message")
    private String message;

    @Option(shortName = 'a', hasValue = false, description = "Add all files")
    private boolean addAll;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Committing: " + message);
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "push", description = "Push to remote")
public class PushCommand implements Command<CommandInvocation> {

    @Option(name = "remote", description = "Remote repository")
    private String remote = "origin";

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Pushing to: " + remote);
        return CommandResult.SUCCESS;
    }
}
```

## Registering Group Commands

Group commands work the same as regular commands - just register them all:

```java
AeshConsoleRunner.builder()
        .command(GitCommand.class)
        .command(CommitCommand.class)
        .command(PushCommand.class)
        .prompt("[git]$ ")
        .addExitCommand()
        .start();
```

## Usage

Users can now invoke commands as:
- `git` - Shows the group command help
- `git commit -m "fix bug"` - Runs the commit command
- `git push --remote upstream` - Runs the push command

## Nested Groups

You can create deeply nested command structures:

```
git
├── commit
├── push
├── pull
└── branch
    ├── create
    ├── delete
    └── list
```

Each level is defined as a separate command with appropriate relationships.
