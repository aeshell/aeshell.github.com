---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Prompt'
weight: 4
---

The `Prompt` class represents the prompt displayed to users before input. It supports simple text prompts, styled prompts with colors, and masked input for passwords.

## Overview

A prompt is displayed each time the shell waits for user input. Æsh Readline provides flexible prompt configuration:

```java
// Simple string prompt
Prompt prompt = new Prompt("$ ");

// Prompt with mask character for passwords
Prompt prompt = new Prompt("Password: ", '*');

// Styled prompt with colors
Prompt prompt = new Prompt(new TerminalString("[user@host]$ ", 
        new TerminalColor(Color.GREEN, Color.DEFAULT)));
```

## Constructors

### Basic Prompt

```java
// Simple text prompt
Prompt prompt = new Prompt("myapp> ");
```

### Prompt with Mask

Used for password input where characters should be hidden:

```java
// Characters are replaced with '*'
Prompt prompt = new Prompt("Password: ", '*');

// Characters are hidden (no echo)
Prompt prompt = new Prompt("Secret: ", '\0');
```

### Prompt with TerminalString

For styled prompts with colors:

```java
TerminalString styledPrompt = new TerminalString(
        "[admin]# ",
        new TerminalColor(Color.RED, Color.DEFAULT),
        CharacterType.BOLD
);
Prompt prompt = new Prompt(styledPrompt);
```

### Prompt with Multiple TerminalStrings

For complex multi-part prompts:

```java
List<TerminalString> parts = Arrays.asList(
        new TerminalString("[", new TerminalColor(Color.WHITE, Color.DEFAULT)),
        new TerminalString("user", new TerminalColor(Color.GREEN, Color.DEFAULT)),
        new TerminalString("@", new TerminalColor(Color.WHITE, Color.DEFAULT)),
        new TerminalString("host", new TerminalColor(Color.BLUE, Color.DEFAULT)),
        new TerminalString("]$ ", new TerminalColor(Color.WHITE, Color.DEFAULT))
);
Prompt prompt = new Prompt(parts);
```

## Prompt Properties

| Property | Type | Description |
|----------|------|-------------|
| `prompt` | `String` | The prompt text |
| `mask` | `Character` | Mask character for input (null if no masking) |
| `ansiString` | `TerminalString` | Styled prompt string |
| `ansiStrings` | `List<TerminalString>` | Multiple styled prompt parts |

## Methods

### getPromptCharacters()

Returns the prompt text without ANSI codes:

```java
Prompt prompt = new Prompt("[user]$ ");
String text = prompt.getPromptCharacters(); // "[user]$ "
```

> **Note:** `getPromptAsString()` is deprecated — use `getPromptCharacters()` instead.

### getMask()

Returns the mask character, or `null` if no masking:

```java
Prompt prompt = new Prompt("Password: ", '*');
Character mask = prompt.getMask(); // '*'
```

### isMasking()

Returns `true` if the prompt uses input masking:

```java
Prompt prompt = new Prompt("Password: ", '*');
boolean masking = prompt.isMasking(); // true
```

### hasANSI()

Returns `true` if the prompt uses ANSI styling:

```java
Prompt prompt = new Prompt(new TerminalString("$ ", Color.GREEN));
boolean hasAnsi = prompt.hasANSI(); // true
```

### copy()

Creates a copy of the prompt:

```java
Prompt original = new Prompt("$ ");
Prompt copy = original.copy();
```

## Prompt Builder

`Prompt.builder()` provides a fluent API for constructing prompts, as an alternative to the telescoping constructors:

```java
// Simple text prompt
Prompt prompt = Prompt.builder()
        .message("$ ")
        .build();

// Password prompt with mask
Prompt prompt = Prompt.builder()
        .message("Password: ")
        .mask('*')
        .build();

// Styled prompt with TerminalString
Prompt prompt = Prompt.builder()
        .terminalString(new TerminalString("[admin]# ",
                new TerminalColor(Color.RED, Color.DEFAULT),
                CharacterType.BOLD))
        .build();

// Prompt with ANSI string
Prompt prompt = Prompt.builder()
        .message("$ ")
        .ansi("\u001B[32m$ \u001B[0m")
        .build();
```

### Builder Methods

| Method | Type | Description |
|--------|------|-------------|
| `message(String)` | `String` | Set the prompt text |
| `ansi(String)` | `String` | Set the ANSI-formatted display string |
| `mask(char)` | `char` | Set mask character for hidden input |
| `promptCodePoints(int[])` | `int[]` | Set prompt as Unicode code points |
| `terminalString(TerminalString)` | `TerminalString` | Set a styled prompt |
| `characters(List<TerminalCharacter>)` | `List` | Set individually formatted characters |

When multiple sources are set, the builder uses this precedence (highest first): `characters`, `terminalString`, `promptCodePoints`, `message`.

## Using Prompts

### With Readline

```java
Readline readline = ReadlineBuilder.builder().build();

Prompt prompt = new Prompt("myapp> ");
readline.readline(connection, prompt, input -> {
    // Handle input
});
```

### With AeshConsoleRunner

```java
AeshConsoleRunner.builder()
        .command(MyCommand.class)
        .prompt("[myshell]$ ")  // Simple string
        .start();

// Or with a Prompt object
Prompt prompt = new Prompt(new TerminalString("$ ", Color.CYAN));
AeshConsoleRunner.builder()
        .command(MyCommand.class)
        .prompt(prompt)
        .start();
```

### Dynamic Prompts

Create prompts that change based on context:

```java
public class DynamicPromptShell {
    private String currentDirectory = "/home/user";
    private String username = "user";
    
    public void start() {
        AeshConsoleRunner.builder()
                .command(CdCommand.class)
                .prompt(this::createPrompt)
                .addExitCommand()
                .start();
    }
    
    private Prompt createPrompt() {
        String promptText = String.format("[%s@%s]$ ", username, currentDirectory);
        return new Prompt(new TerminalString(promptText, Color.GREEN));
    }
    
    public void setCurrentDirectory(String dir) {
        this.currentDirectory = dir;
    }
}
```

## Styled Prompts

### Colors

```java
// Foreground color only
new TerminalString("$ ", new TerminalColor(Color.GREEN, Color.DEFAULT))

// Foreground and background
new TerminalString("$ ", new TerminalColor(Color.WHITE, Color.BLUE))

// Using RGB colors (24-bit)
new TerminalString("$ ", new TerminalColor(new Color(255, 100, 50), Color.DEFAULT))
```

### Text Styles

```java
// Bold text
new TerminalString("$ ", Color.DEFAULT, CharacterType.BOLD)

// Italic text
new TerminalString("$ ", Color.DEFAULT, CharacterType.ITALIC)

// Underlined text
new TerminalString("$ ", Color.DEFAULT, CharacterType.UNDERLINE)

// Combined styles
new TerminalString("$ ", 
        new TerminalColor(Color.RED, Color.DEFAULT),
        CharacterType.BOLD,
        CharacterType.UNDERLINE)
```

### Multi-Part Styled Prompts

```java
List<TerminalString> promptParts = Arrays.asList(
        // Username in green
        new TerminalString(username, new TerminalColor(Color.GREEN, Color.DEFAULT)),
        // Separator
        new TerminalString(":", Color.DEFAULT),
        // Directory in blue
        new TerminalString(currentDir, new TerminalColor(Color.BLUE, Color.DEFAULT), CharacterType.BOLD),
        // Prompt symbol based on user type
        new TerminalString(isRoot ? "# " : "$ ", Color.DEFAULT)
);
Prompt prompt = new Prompt(promptParts);
```

## Password Input

### Basic Password Prompt

```java
// Reading password in a command
@Override
public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
    Shell shell = invocation.getShell();
    
    // Password input with asterisk mask
    String password = shell.readLine(new Prompt("Password: ", '*'));
    
    // Authenticate...
    return CommandResult.SUCCESS;
}
```

### Hidden Input (No Echo)

```java
// Use null character for completely hidden input
Prompt hiddenPrompt = new Prompt("API Key: ", '\0');
String apiKey = shell.readLine(hiddenPrompt);
```

### Confirmation Pattern

```java
@Override
public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
    Shell shell = invocation.getShell();
    
    // Get password
    String password = shell.readLine(new Prompt("Enter new password: ", '*'));
    String confirm = shell.readLine(new Prompt("Confirm password: ", '*'));
    
    if (!password.equals(confirm)) {
        invocation.println("Passwords do not match!");
        return CommandResult.FAILURE;
    }
    
    // Save password...
    invocation.println("Password updated successfully.");
    return CommandResult.SUCCESS;
}
```

## Complete Example

```java
import org.aesh.readline.*;
import org.aesh.readline.terminal.formatting.*;
import org.aesh.terminal.Connection;
import java.util.Arrays;
import java.util.List;

public class StyledPromptExample {
    
    public static void main(String[] args) {
        Connection connection = /* obtain connection */;
        Readline readline = ReadlineBuilder.builder().build();
        
        // Create a fancy prompt
        Prompt prompt = createFancyPrompt("user", "/home/user");
        
        readline.readline(connection, prompt, input -> {
            if (input != null && !input.trim().isEmpty()) {
                System.out.println("You entered: " + input);
            }
            
            // Continue reading
            readline.readline(connection, prompt, this);
        });
    }
    
    private static Prompt createFancyPrompt(String user, String directory) {
        List<TerminalString> parts = Arrays.asList(
                // Opening bracket
                new TerminalString("[", 
                        new TerminalColor(Color.WHITE, Color.DEFAULT)),
                
                // Username in green bold
                new TerminalString(user, 
                        new TerminalColor(Color.GREEN, Color.DEFAULT),
                        CharacterType.BOLD),
                
                // @ symbol
                new TerminalString("@", 
                        new TerminalColor(Color.WHITE, Color.DEFAULT)),
                
                // Hostname in cyan
                new TerminalString("localhost", 
                        new TerminalColor(Color.CYAN, Color.DEFAULT)),
                
                // Space and directory in blue
                new TerminalString(" " + shortenPath(directory), 
                        new TerminalColor(Color.BLUE, Color.DEFAULT)),
                
                // Closing bracket and prompt symbol
                new TerminalString("]$ ", 
                        new TerminalColor(Color.WHITE, Color.DEFAULT))
        );
        
        return new Prompt(parts);
    }
    
    private static String shortenPath(String path) {
        // Replace home directory with ~
        String home = System.getProperty("user.home");
        if (path.startsWith(home)) {
            return "~" + path.substring(home.length());
        }
        return path;
    }
}
```

## Git-Style Prompt

Create a prompt that shows git branch information:

```java
public class GitPrompt {
    
    public Prompt create() {
        String branch = getGitBranch();
        String directory = getCurrentDirectory();
        
        List<TerminalString> parts = new ArrayList<>();
        
        // Directory in blue
        parts.add(new TerminalString(directory + " ", 
                new TerminalColor(Color.BLUE, Color.DEFAULT)));
        
        // Git branch if available
        if (branch != null) {
            parts.add(new TerminalString("(", Color.DEFAULT));
            parts.add(new TerminalString(branch, 
                    new TerminalColor(Color.MAGENTA, Color.DEFAULT)));
            parts.add(new TerminalString(") ", Color.DEFAULT));
        }
        
        // Prompt symbol
        parts.add(new TerminalString("$ ", Color.DEFAULT));
        
        return new Prompt(parts);
    }
    
    private String getGitBranch() {
        try {
            Process process = Runtime.getRuntime()
                    .exec(new String[]{"git", "branch", "--show-current"});
            BufferedReader reader = new BufferedReader(
                    new InputStreamReader(process.getInputStream()));
            String branch = reader.readLine();
            process.waitFor();
            return process.exitValue() == 0 ? branch : null;
        } catch (Exception e) {
            return null;
        }
    }
    
    private String getCurrentDirectory() {
        return System.getProperty("user.dir");
    }
}
```

## Best Practices

1. **Keep prompts short** - Long prompts reduce the space available for user input.

2. **Use colors meaningfully** - Use colors to convey information (e.g., green for normal, red for root).

3. **Always mask passwords** - Never echo password input to the terminal.

4. **Test color fallbacks** - Ensure prompts are readable when colors are not supported.

5. **Consider accessibility** - Don't rely solely on color to convey information.

```java
// Good: Information conveyed by text AND color
new TerminalString("[admin]# ", new TerminalColor(Color.RED, Color.DEFAULT))

// Bad: Information only conveyed by color
new TerminalString("$ ", new TerminalColor(Color.RED, Color.DEFAULT))
```

6. **Use dynamic prompts sparingly** - Regenerating complex prompts on every input can impact performance.
