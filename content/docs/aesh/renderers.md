---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Renderers'
---

Renderers customize how option values are displayed in help text and output.

## OptionRenderer Interface

```java
public interface OptionRenderer<T extends ParsedOption> {
    String render(T option);
}
```

## Custom Value Display

Customize how sensitive values are displayed:

```java
public class PasswordRenderer implements OptionRenderer<ProcessedOption> {

    @Override
    public String render(ProcessedOption option) {
        return "******** (password hidden)";
    }
}
```

Usage:

```java
@CommandDefinition(name = "login")
public class LoginCommand implements Command<CommandInvocation> {

    @Option(
        name = "username",
        description = "Username"
    )
    private String username;

    @Option(
        name = "password",
        renderer = PasswordRenderer.class,
        description = "Password"
    )
    private String password;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Password is displayed normally here, only hidden in help
        return CommandResult.SUCCESS;
    }
}
```

## Colorized Output

```java
public class ColorizedRenderer implements OptionRenderer<ProcessedOption> {

    @Override
    public String render(ProcessedOption option) {
        TerminalString colorized = new TerminalString(
            option.getValue(),
            TerminalColor.DEFAULT_TEXT,
            TerminalTextStyle.INTENSITY_BOLD
        );
        return colorized.toString();
    }
}
```

## Conditional Rendering

```java
public class SmartPathRenderer implements OptionRenderer<ProcessedOption> {

    @Override
    public String render(ProcessedOption option) {
        String value = option.getValue();
        // Show relative paths, abbreviate long paths
        Path path = Paths.get(value);
        if (path.isAbsolute() && path.getNameCount() > 3) {
            return "..." + path.subpath(path.getNameCount() - 3, path.getNameCount());
        }
        return value;
    }
}
```

## Default Value Display

Render default values in a specific format:

```java
public class DefaultValueRenderer implements OptionRenderer<ProcessedOption> {

    @Override
    public String render(ProcessedOption option) {
        String value = option.getValue();
        String defaultValue = option.getDefaultValues().length > 0 
            ? option.getDefaultValues()[0] 
            : null;
        
        if (value != null && value.equals(defaultValue)) {
            return value + " (default)";
        }
        return value != null ? value : "not set";
    }
}
```
