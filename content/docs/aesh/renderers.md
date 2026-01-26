---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Renderers'
weight: 12
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
import org.aesh.readline.terminal.formatting.TerminalColor;
import org.aesh.readline.terminal.formatting.TerminalString;
import org.aesh.readline.terminal.formatting.TerminalTextStyle;
import org.aesh.readline.terminal.formatting.CharacterType;
import org.aesh.readline.terminal.formatting.Color;

public class ColorizedRenderer implements OptionRenderer<ProcessedOption> {

    @Override
    public String render(ProcessedOption option) {
        TerminalString colorized = new TerminalString(
            option.getValue(),
            new TerminalColor(Color.CYAN, Color.DEFAULT),
            new TerminalTextStyle(CharacterType.BOLD)
        );
        return colorized.toString();
    }
}
```

### Theme-Aware Rendering

For colors that adapt to the terminal's light or dark theme, use the semantic color methods with detected terminal capabilities:

```java
import org.aesh.readline.terminal.TerminalColorDetector;
import org.aesh.readline.terminal.formatting.TerminalColor;
import org.aesh.readline.terminal.formatting.TerminalString;
import org.aesh.terminal.utils.TerminalColorCapability;

public class AdaptiveRenderer implements OptionRenderer<ProcessedOption> {
    private final TerminalColorCapability capability;
    
    public AdaptiveRenderer(TerminalColorCapability capability) {
        this.capability = capability;
    }

    @Override
    public String render(ProcessedOption option) {
        // Use theme-aware colors that work on both light and dark backgrounds
        TerminalColor color = TerminalColor.forHighlight(capability);
        return new TerminalString(option.getValue(), color).toString();
    }
}
```

The semantic color methods automatically choose appropriate brightness:

| Method | Dark Theme | Light Theme | Use For |
|--------|------------|-------------|---------|
| `forError()` | Bright red | Normal red | Error messages |
| `forSuccess()` | Bright green | Normal green | Success confirmations |
| `forWarning()` | Bright yellow | Normal yellow | Warnings |
| `forInfo()` | Bright cyan | Normal blue | Informational text |
| `forHighlight()` | Bright white | Black | Emphasized text |
| `forMuted()` | Normal white | Normal black | Secondary text |

See the Readline [Terminal Colors](/docs/readline/terminal-colors) documentation for more details on RGB colors and color depth adaptation.

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
