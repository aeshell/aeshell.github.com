---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Selectors'
weight: 11
---

Selectors provide interactive input prompts that go beyond simple text entry. They let users type text, enter passwords with masking, choose one item from a list, or select multiple items with checkboxes -- all driven by arrow keys and rendered directly in the terminal.

## SelectorType

| Type | Description | User Interaction |
|------|-------------|------------------|
| `INPUT` | Text input prompt | Type text, press Enter |
| `PASSWORD` | Masked password input | Type text (shown as `*`), press Enter |
| `SELECT` | Single selection from a list | Arrow keys to move, Enter to select |
| `SELECTIONS` | Multiple selection from a list | Arrow keys to move, Space to toggle, Enter to finish |
| `NO_OP` | No selector (default) | Standard option parsing |

## Using Selectors with @Option

The `selector` property on `@Option` integrates selectors directly into option parsing. When an option has a selector and is flagged with `askIfNotSet = true`, the user is prompted interactively if the value was not provided on the command line.

### Text Input

```java
@CommandDefinition(name = "greet", description = "Greet someone")
public class GreetCommand implements Command<CommandInvocation> {

    @Option(selector = SelectorType.INPUT, askIfNotSet = true,
            description = "Name to greet")
    private String name;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name + "!");
        return CommandResult.SUCCESS;
    }
}
```

If the user runs `greet` without `--name`, they see a text prompt:

```
Name to greet: _
```

### Password Input

```java
@CommandDefinition(name = "login", description = "Log in")
public class LoginCommand implements Command<CommandInvocation> {

    @Option(description = "Username", required = true)
    private String username;

    @Option(selector = SelectorType.PASSWORD, askIfNotSet = true,
            description = "Password")
    private String password;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Logging in as " + username);
        // authenticate...
        return CommandResult.SUCCESS;
    }
}
```

If `--password` is omitted, the user is prompted with masked input:

```
Password: ****
```

### Single Selection

Provide a list of choices using `defaultValue`. The user navigates with arrow keys and confirms with Enter:

```java
@CommandDefinition(name = "init", description = "Initialize project")
public class InitCommand implements Command<CommandInvocation> {

    @Option(selector = SelectorType.SELECT, askIfNotSet = true,
            defaultValue = {"java", "kotlin", "scala", "groovy"},
            description = "Project language")
    private String language;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Initializing " + language + " project");
        return CommandResult.SUCCESS;
    }
}
```

If `--language` is omitted, the user sees an interactive list:

```
Project language  [Use arrow up/down to move and enter/space to select]
> java
  kotlin
  scala
  groovy
```

The `>` indicator moves with the arrow keys. Press Enter or Space to confirm.

### Multiple Selection

Similar to single selection, but the user can toggle multiple items with Space before confirming with Enter:

```java
@CommandDefinition(name = "setup", description = "Setup project features")
public class SetupCommand implements Command<CommandInvocation> {

    @Option(selector = SelectorType.SELECTIONS, askIfNotSet = true,
            defaultValue = {"logging", "metrics", "security", "caching"},
            description = "Features to enable")
    private String features;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Enabled: " + features);
        return CommandResult.SUCCESS;
    }
}
```

If omitted, the user sees a multi-select list:

```
Features to enable  [Use arrow up/down to move and space to select. Enter to finish]
> [ ] logging
  [ ] metrics
  [ ] security
  [ ] caching
```

Use Space to toggle checkboxes, Enter to confirm. Selected items are marked with `[x]`. When the list exceeds the terminal height, pagination is handled automatically with the alternate screen buffer.

## Using Selectors Programmatically

You can also use the `Selector` class directly in your command's `execute()` method for full control:

```java
import org.aesh.selector.Selector;
import org.aesh.selector.SelectorType;

@CommandDefinition(name = "deploy", description = "Deploy application")
public class DeployCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation)
            throws InterruptedException {
        Shell shell = invocation.getShell();

        // Single selection
        Selector envSelector = new Selector(
            SelectorType.SELECT,
            new String[]{"development", "staging", "production"},
            "Target environment?"
        );
        List<String> chosen = envSelector.doSelect(shell);
        String env = chosen.get(0);

        // Confirmation via text input
        Selector confirmSelector = new Selector(
            SelectorType.INPUT,
            new String[]{},
            "Type '" + env + "' to confirm:"
        );
        List<String> confirmation = confirmSelector.doSelect(shell);

        if (env.equals(confirmation.get(0))) {
            invocation.println("Deploying to " + env);
        } else {
            invocation.println("Aborted.");
        }

        return CommandResult.SUCCESS;
    }
}
```

## Multi-Step Wizards

Combine multiple selectors to build interactive setup wizards:

```java
@CommandDefinition(name = "new-project", description = "Create a new project")
public class NewProjectCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation)
            throws InterruptedException {
        Shell shell = invocation.getShell();

        // Step 1: Project name
        Selector nameInput = new Selector(SelectorType.INPUT,
            new String[]{}, "Project name:");
        String name = nameInput.doSelect(shell).get(0);

        // Step 2: Language
        Selector langSelect = new Selector(SelectorType.SELECT,
            new String[]{"java", "kotlin", "scala"},
            "Language:");
        String lang = langSelect.doSelect(shell).get(0);

        // Step 3: Features (multi-select)
        Selector featureSelect = new Selector(SelectorType.SELECTIONS,
            new String[]{"tests", "ci", "docker", "docs"},
            "Features:");
        List<String> features = featureSelect.doSelect(shell);

        invocation.println("Creating " + lang + " project '" + name
            + "' with: " + String.join(", ", features));
        return CommandResult.SUCCESS;
    }
}
```
