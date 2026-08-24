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

## Multi-Line Prompts

Prompts can span multiple lines, with contextual information on upper lines and the input cursor on the last line — similar to [starship](https://starship.rs/) or [powerlevel10k](https://github.com/romkatv/powerlevel10k).

```
myapp on main via ☕ v21
❯ user types here_
```

### Creating Multi-Line Prompts

Use `\n` in the prompt string:

```java
Prompt prompt = new Prompt("myapp on main via ☕ v21\n❯ ");
```

Or use the builder with `line()`:

```java
Prompt prompt = Prompt.builder()
    .line("myapp on main via ☕ v21")
    .line("❯ ")
    .build();
```

### With ANSI Colors

Each line's ANSI codes are detected independently:

```java
Prompt prompt = new Prompt(
    "\033[36mmyapp\033[0m on \033[32mmain\033[0m\n\033[1m❯\033[0m ");
```

Or with the builder:

```java
Prompt prompt = Prompt.builder()
    .line("\033[36mmyapp\033[0m on \033[32mmain\033[0m via ☕ v21")
    .line("\033[1m❯\033[0m ")
    .rightPrompt("3.2s")
    .build();
```

### Powerlevel10k Style

```java
Prompt prompt = Prompt.builder()
    .line("┌─[myapp]─[git:main]─[took 3.2s]")
    .line("└─$ ")
    .build();
```

### How It Works

- `length()` returns the visible length of the **last line only** — this is what matters for cursor positioning
- `lineCount()` returns the number of lines in the prompt
- Buffer cursor math operates relative to the last line
- Upper prompt lines are rendered but don't affect cursor calculations
- Right prompt aligns with the last line
- `printAbove()` and status lines account for the extra prompt lines automatically

## Continuation Prompt

When the user enters a line ending with a backslash (`\`) or containing an unclosed quote, readline continues reading on the next line with a **continuation prompt**. This is equivalent to `PS2` in bash/zsh.

By default, the continuation prompt is `"> "`. You can customize it via `ReadlineRequest.builder()`:

```java
ReadlineRequest.builder()
    .connection(conn)
    .prompt(new Prompt("$ "))
    .continuationPrompt("... ")
    .requestHandler(line -> { /* ... */ })
    .build();
```

Example session with the default continuation prompt:

```
$ echo "hello \
> world"
hello world
```

With a custom continuation prompt:

```
$ echo "hello \
... world"
hello world
```

The continuation prompt also accepts a `Prompt` object for styled prompts:

```java
ReadlineRequest.builder()
    .connection(conn)
    .prompt(new Prompt("$ "))
    .continuationPrompt(new Prompt(
        new TerminalString("... ", new TerminalColor(Color.YELLOW, Color.DEFAULT))))
    .requestHandler(line -> { /* ... */ })
    .build();
```

## ANSI Auto-Detection

When creating a prompt with embedded ANSI escape codes, the visible length is calculated automatically:

```java
// ANSI codes are detected — visible length is 2 ("$ "), not 11
Prompt prompt = new Prompt("\033[32m$\033[0m ");

// Equivalent to the two-arg constructor:
Prompt prompt = new Prompt("$ ", "\033[32m$\033[0m ");
```

This prevents cursor positioning issues that occur when the prompt length includes invisible escape sequences.

## Constructors

### Basic Prompt

```java
// Simple text prompt
Prompt prompt = new Prompt("myapp> ");

// With ANSI codes — length auto-detected
Prompt prompt = new Prompt("\033[1;31mERROR>\033[0m ");
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

> **Note:** `getPromptAsString()` has been removed — use `getPromptCharacters()` instead.

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

Prompts can change dynamically between readline cycles using `promptSupplier`. The supplier is called once at the start of each `readline()` invocation:

```java
// Readline-level: dynamic prompt via ReadlineRequest
readline.readline(
    ReadlineRequest.builder()
        .connection(conn)
        .promptSupplier(() -> {
            String branch = getGitBranch();
            return Prompt.builder()
                .line("myapp on " + branch)
                .line("❯ ")
                .build();
        })
        .requestHandler(line -> { /* ... */ })
        .build());
```

This is useful for starship-style prompts that show dynamic information:

```java
// Show command execution time in prompt
AtomicLong lastDuration = new AtomicLong();

ReadlineRequest.builder()
    .connection(conn)
    .promptSupplier(() -> {
        long ms = lastDuration.getAndSet(0);
        if (ms > 2000) {
            return Prompt.builder()
                .line("myapp took " + String.format("%.1fs", ms / 1000.0))
                .line("❯ ")
                .build();
        }
        return new Prompt("❯ ");
    })
    .requestHandler(line -> { /* ... */ })
    .build();
```

If both `prompt` and `promptSupplier` are set, the supplier takes precedence.

#### Aesh-Level Dynamic Prompts

In Æsh, create prompts that change based on context:

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
