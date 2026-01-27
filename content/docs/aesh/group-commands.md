---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Group Commands'
weight: 7
---

Group commands allow you to create hierarchical command structures, similar to `git` with subcommands like `commit`, `push`, `pull`.

## @GroupCommandDefinition

Use the `@GroupCommandDefinition` annotation to define a command group. The `groupCommands` property specifies which commands are subcommands of this group:

```java
@GroupCommandDefinition(
    name = "git",
    description = "Version control system",
    groupCommands = {CommitCommand.class, PushCommand.class, PullCommand.class}
)
public class GitCommand implements GroupCommand<CommandInvocation> {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // This is called when user types just "git" without a subcommand
        invocation.println("Usage: git <command> [options]");
        invocation.println("Commands: commit, push, pull");
        return CommandResult.SUCCESS;
    }
}
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | required | The group command name |
| `description` | `String` | `""` | Description shown in help |
| `groupCommands` | `Class[]` | `{}` | Array of subcommand classes |
| `generateHelp` | `boolean` | `false` | Auto-generate `--help` option |
| `aliases` | `String[]` | `{}` | Alternative names for the group |

## Subcommands

Define subcommands as regular commands with `@CommandDefinition`:

```java
@CommandDefinition(name = "commit", description = "Commit changes")
public class CommitCommand implements Command<CommandInvocation> {

    @Option(shortName = 'm', description = "Commit message", required = true)
    private String message;

    @Option(shortName = 'a', hasValue = false, description = "Add all files")
    private boolean addAll;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (addAll) {
            invocation.println("Adding all modified files...");
        }
        invocation.println("Committing: " + message);
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "push", description = "Push to remote")
public class PushCommand implements Command<CommandInvocation> {

    @Option(name = "remote", description = "Remote repository")
    private String remote = "origin";

    @Option(name = "branch", description = "Branch to push")
    private String branch = "main";

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Pushing " + branch + " to " + remote + "...");
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "pull", description = "Pull from remote")
public class PullCommand implements Command<CommandInvocation> {

    @Option(name = "remote", description = "Remote repository")
    private String remote = "origin";

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Pulling from " + remote + "...");
        return CommandResult.SUCCESS;
    }
}
```

## Registering Group Commands

Only the top-level group command needs to be registered. The subcommands are automatically available through the `groupCommands` property:

```java
AeshConsoleRunner.builder()
        .command(GitCommand.class)  // Subcommands are included via @GroupCommandDefinition
        .prompt("[myapp]$ ")
        .addExitCommand()
        .start();
```

## Usage

Users can now invoke commands as:
- `git` - Shows the group command help/usage
- `git commit -m "fix bug"` - Runs the commit command
- `git commit -a -m "fix all"` - Commit with add all option
- `git push --remote upstream` - Runs the push command
- `git pull` - Runs the pull command with defaults

```
[myapp]$ git
Usage: git <command> [options]
Commands: commit, push, pull

[myapp]$ git commit -m "Initial commit"
Committing: Initial commit

[myapp]$ git push --remote upstream --branch develop
Pushing develop to upstream...
```

## Nested Groups

You can create deeply nested command structures by having a group command as a subcommand of another group:

```
docker
├── container
│   ├── ls
│   ├── start
│   ├── stop
│   └── rm
├── image
│   ├── ls
│   ├── pull
│   └── rm
└── volume
    ├── create
    └── rm
```

### Example: Docker-like Command Structure

```java
// Top-level group
@GroupCommandDefinition(
    name = "docker",
    description = "Container management",
    groupCommands = {ContainerGroup.class, ImageGroup.class, VolumeGroup.class}
)
public class DockerCommand implements GroupCommand<CommandInvocation> {
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Usage: docker <command>");
        invocation.println("Commands: container, image, volume");
        return CommandResult.SUCCESS;
    }
}

// Nested group for container commands
@GroupCommandDefinition(
    name = "container",
    description = "Manage containers",
    groupCommands = {ContainerLsCommand.class, ContainerStartCommand.class, 
                     ContainerStopCommand.class, ContainerRmCommand.class}
)
public class ContainerGroup implements GroupCommand<CommandInvocation> {
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Usage: docker container <command>");
        return CommandResult.SUCCESS;
    }
}

// Leaf command
@CommandDefinition(name = "ls", description = "List containers")
public class ContainerLsCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'a', hasValue = false, description = "Show all containers")
    private boolean all;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (all) {
            invocation.println("Listing all containers...");
        } else {
            invocation.println("Listing running containers...");
        }
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "start", description = "Start a container")
public class ContainerStartCommand implements Command<CommandInvocation> {
    
    @Argument(description = "Container ID or name", required = true)
    private String container;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Starting container: " + container);
        return CommandResult.SUCCESS;
    }
}
```

Usage:
```
[myapp]$ docker container ls -a
Listing all containers...

[myapp]$ docker container start mycontainer
Starting container: mycontainer
```

## Group Commands with Shared Options

You can add options to the group command itself that apply to all subcommands:

```java
@GroupCommandDefinition(
    name = "kubectl",
    description = "Kubernetes CLI",
    groupCommands = {GetCommand.class, ApplyCommand.class, DeleteCommand.class}
)
public class KubectlCommand implements GroupCommand<CommandInvocation> {
    
    @Option(shortName = 'n', name = "namespace", description = "Kubernetes namespace")
    private String namespace = "default";
    
    @Option(name = "context", description = "Kubernetes context")
    private String context;
    
    public String getNamespace() {
        return namespace;
    }
    
    public String getContext() {
        return context;
    }
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Usage: kubectl [options] <command>");
        return CommandResult.SUCCESS;
    }
}
```

## Best Practices

1. **Use meaningful group names** - Group names should clearly indicate the category of commands they contain.

2. **Provide help in the group command** - When the group command is invoked without a subcommand, show usage information.

3. **Keep nesting shallow** - Avoid more than 2-3 levels of nesting for usability.

4. **Use consistent naming** - Follow conventions like `noun-verb` or `verb-noun` consistently.

5. **Generate help** - Use `generateHelp = true` for automatic help generation.
