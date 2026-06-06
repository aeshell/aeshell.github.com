---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Group Commands'
weight: 7
---

Group commands allow you to create hierarchical command structures, similar to `git` with subcommands like `commit`, `push`, `pull`.

## Defining a Group Command

Use the `groupCommands` property on `@CommandDefinition` to define subcommands:

```java
@CommandDefinition(
    name = "git",
    description = "Version control system",
    groupCommands = {CommitCommand.class, PushCommand.class, PullCommand.class}
)
public class GitCommand implements Command<CommandInvocation> {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // This is called when user types just "git" without a subcommand
        invocation.println("Usage: git <command> [options]");
        invocation.println("Commands: commit, push, pull");
        return CommandResult.SUCCESS;
    }
}
```

A command becomes a group command simply by having a non-empty `groupCommands` array. All standard `@CommandDefinition` properties are available (`generateHelp`, `version`, `helpUrl`, `sortOptions`, etc.).

{{< callout type="info" >}}
The `@GroupCommandDefinition` annotation is deprecated. Use `@CommandDefinition` with `groupCommands` instead. Existing code using `@GroupCommandDefinition` continues to work.
{{< /callout >}}

## Subcommands

Define subcommands as regular commands with `@CommandDefinition`. Subcommands support `aliases` for short alternative names:

```java
@CommandDefinition(name = "commit", aliases = {"ci"}, description = "Commit changes")
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
        .command(GitCommand.class)  // Subcommands are included via groupCommands
        .prompt("[myapp]$ ")
        .addExitCommand()
        .start();
```

Subcommand aliases work for both direct invocation and shell completion:

```bash
git commit -m "fix"   # primary name
git ci -m "fix"       # alias — resolves to commit
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
@CommandDefinition(
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
@CommandDefinition(
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

You can add options to the group command itself. These options are parsed before the subcommand name:

```java
@CommandDefinition(
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

### Inherited Options

To make a group option available to subcommands as well, add `inherited = true`. The option can then be placed either before or after the subcommand name, and its value is automatically propagated to matching fields on the child command:

```java
@CommandDefinition(
    name = "cli",
    description = "My CLI tool",
    groupCommands = {RunCommand.class, TestCommand.class}
)
public class CliCommand implements Command<CommandInvocation> {

    @Option(hasValue = false, inherited = true, description = "Enable verbose output")
    private boolean verbose;

    @Option(inherited = true, description = "Config file path")
    private String config;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "run", description = "Run application")
public class RunCommand implements Command<CommandInvocation> {

    // Fields matching parent's inherited options -- auto-populated
    private boolean verbose;
    private String config;

    @Argument(description = "Main class")
    private String mainClass;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (verbose) {
            invocation.println("Config: " + config);
        }
        invocation.println("Running " + mainClass);
        return CommandResult.SUCCESS;
    }
}
```

Both of these are equivalent:

```bash
$ cli --verbose --config app.yml run MyApp
$ cli run --verbose --config app.yml MyApp
```

See [Options - Inherited Options](/docs/aesh/options#inherited-options) for full details.

## Sub-Command Mode

Group commands can enter an interactive **sub-command mode** where users work within the group's context. This is useful for workflows that involve multiple operations on the same resource.

```java
@CommandDefinition(
    name = "project",
    description = "Project management",
    groupCommands = {BuildCommand.class, TestCommand.class}
)
public class ProjectCommand implements Command<CommandInvocation> {

    @Option(name = "name", required = true)
    private String projectName;

    @Option(name = "verbose", hasValue = false, inherited = true)  // Available to subcommands
    private boolean verbose;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Project: " + projectName);

        // Enter sub-command mode - prompt changes to "project[myapp]>"
        invocation.enterSubCommandMode(this);

        return CommandResult.SUCCESS;
    }
}
```

Usage:
```
[myapp]$ project --name=myapp --verbose
Project: myapp
Entering project mode.

project[myapp]> build
Building myapp...

project[myapp]> test
Testing myapp...

project[myapp]> exit
[myapp]$
```

See [Sub-Command Mode](/docs/aesh/sub-command-mode) for complete documentation on:
- Accessing parent values via `@ParentCommand` or `getParentValue()`
- Inherited options with `inherited = true`
- Configuring prompts, exit commands, and messages
- Nested contexts

## Querying Subcommand Names

Use `getSubcommandNames()` on the registry to get the names of all subcommands for a group command. This avoids hardcoding subcommand names when you need to distinguish them from positional arguments:

```java
CommandRegistry<CommandInvocation> registry = AeshCommandRegistryBuilder.builder()
        .command(GitCommand.class)
        .create();

Set<String> subcommands = registry.getSubcommandNames("git");
// returns {"commit", "push", "pull"}
```

See [Command Registry - getSubcommandNames](/docs/aesh/command-registry#getsubcommandnamesstring-parentcommandname) for more details.

## Best Practices

1. **Use meaningful group names** - Group names should clearly indicate the category of commands they contain.

2. **Provide help in the group command** - When the group command is invoked without a subcommand, show usage information.

3. **Keep nesting shallow** - Avoid more than 2-3 levels of nesting for usability.

4. **Use consistent naming** - Follow conventions like `noun-verb` or `verb-noun` consistently.

5. **Generate help** - Use `generateHelp = true` for automatic help generation.

6. **Consider sub-command mode** - For group commands that users will interact with repeatedly, enable sub-command mode for a better experience.
