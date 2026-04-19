---
date: '2026-01-12T11:00:00+01:00'
draft: false
title: 'Console and Runtime Runners'
weight: 13
---

Æsh provides two main runner classes for executing commands: `AeshConsoleRunner` for interactive console applications and `AeshRuntimeRunner` for programmatic command execution.

## AeshConsoleRunner

`AeshConsoleRunner` creates an interactive console with a read-eval-print loop (REPL). It's ideal for building CLI applications where users type commands interactively.

### Basic Usage

```java
import org.aesh.AeshConsoleRunner;

public class InteractiveShell {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(MyCommand.class)
                .prompt("[myshell]$ ")
                .addExitCommand()
                .start();
    }
}
```

### Builder API

The `AeshConsoleRunner.builder()` provides a fluent API for configuration:

#### Command Registration

```java
AeshConsoleRunner.builder()
    // Register a single command class
    .command(HelloCommand.class)
    
    // Register multiple commands
    .command(GreetCommand.class)
    .command(ExitCommand.class)
    
    // Register a command instance
    .command(new CustomCommand())
    
    // Register commands from a registry
    .commandRegistry(myCommandRegistry)
```

#### Prompt Configuration

```java
AeshConsoleRunner.builder()
    // Simple string prompt
    .prompt("[myapp]$ ")
    
    // Dynamic prompt using a Prompt object
    .prompt(new Prompt("[" + getCurrentDirectory() + "]$ "))
```

#### Exit Command

```java
AeshConsoleRunner.builder()
    // Add default exit command (responds to "exit" and "quit")
    .addExitCommand()
    
    // The exit command allows users to exit the shell gracefully
```

#### Settings Configuration

```java
import org.aesh.terminal.tty.Settings;

AeshConsoleRunner.builder()
    .settings(Settings.builder()
        .enableAlias(true)
        .historyFile("/path/to/.history")
        .historySize(500)
        .logging(true)
        .enableExport(true)
        .build())
```

### Complete Example

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;

@CommandDefinition(name = "echo", description = "Echo text to output")
class EchoCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'n', hasValue = false, description = "Do not output trailing newline")
    private boolean noNewline;
    
    @Option(description = "Text to echo")
    private String text;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (text != null) {
            if (noNewline) {
                invocation.print(text);
            } else {
                invocation.println(text);
            }
        }
        return CommandResult.SUCCESS;
    }
}

public class MyConsole {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(EchoCommand.class)
                .prompt("[console]$ ")
                .addExitCommand()
                .start();
    }
}
```

### Lifecycle Methods

Once started, the console runs until:
- The user executes an exit command
- The program is interrupted (Ctrl+C)
- An error occurs

The `start()` method blocks until the console exits.

## AeshRuntimeRunner

`AeshRuntimeRunner` executes commands programmatically without an interactive console. It's useful for scripting, testing, or executing commands based on application logic.

### Basic Usage

```java
import org.aesh.AeshRuntimeRunner;

public class ProgrammaticExecution {
    public static void main(String[] args) {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(MyCommand.class)
                .build();
        
        // Execute a command
        String result = runner.execute("mycommand --option value arg1");
        System.out.println(result);
    }
}
```

### Builder API

Similar to `AeshConsoleRunner`, but without interactive features:

```java
AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
    // Register commands
    .command(Command1.class)
    .command(Command2.class)
    
    // Configure settings
    .settings(Settings.builder()
        .enableAlias(true)
        .enableExport(true)
        .build())
    
    .build();
```

### Executing Commands

#### Single Command Execution

```java
// Execute and get the output as a string
String output = runner.execute("greet --name Alice");

// Execute with error handling
try {
    String output = runner.execute("command --invalid-option");
    System.out.println(output);
} catch (Exception e) {
    System.err.println("Command failed: " + e.getMessage());
}
```

#### Multiple Command Execution

```java
// Execute multiple commands sequentially
runner.execute("command1 arg1");
runner.execute("command2 --option value");
runner.execute("command3");
```

#### Capturing Output

The runner captures all output from `CommandInvocation.println()` and `CommandInvocation.print()`:

```java
@CommandDefinition(name = "info", description = "Display info")
class InfoCommand implements Command<CommandInvocation> {
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("System Information:");
        invocation.println("Version: 1.0");
        invocation.println("Status: Running");
        return CommandResult.SUCCESS;
    }
}

// Later in code:
String info = runner.execute("info");
// info contains:
// System Information:
// Version: 1.0
// Status: Running
```

### Reading Command-Line Arguments

`AeshRuntimeRunner` can read command-line arguments directly using the `.args()` method, making it easy to create CLI tools:

```java
import org.aesh.AeshRuntimeRunner;
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;

@CommandDefinition(name = "process", description = "Process data")
class ProcessCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'f', description = "Input file")
    private String file;
    
    @Option(shortName = 'v', hasValue = false, description = "Verbose output")
    private boolean verbose;
    
    @Option(shortName = 'o', description = "Output format")
    private String format = "json";
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Processing file: " + file);
        invocation.println("Output format: " + format);
        
        if (verbose) {
            invocation.println("Verbose mode enabled");
        }
        
        // Process the file...
        
        return CommandResult.SUCCESS;
    }
}

public class CliTool {
    public static void main(String[] args) {
        // Pass command-line args directly and execute
        AeshRuntimeRunner.builder()
                .command(ProcessCommand.class)
                .args(args)
                .execute();
    }
}
```

Run this from the command line:

```bash
java -jar mytool.jar process -f input.txt -o xml -v
```

Output:
```
Processing file: input.txt
Output format: xml
Verbose mode enabled
```

The `.args(args)` method automatically parses the command-line arguments and passes them to the registered command. When combined with `.execute()`, it provides a complete single-call execution pattern.

#### Arguments with Special Characters

Arguments containing spaces, embedded quotes, backslashes, newlines, or shell operator characters (`|`, `;`, `>`) are fully supported. The args are passed directly to the command parser without string reconstruction or re-parsing, so their content is preserved exactly:

```java
@CommandDefinition(name = "run", description = "Run code")
class RunCommand implements Command<CommandInvocation> {

    @Option(shortName = 'c', description = "Code to execute")
    private String code;

    @Argument(description = "First positional argument")
    private String firstArg;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Code: " + code);
        invocation.println("Arg: " + firstArg);
        return CommandResult.SUCCESS;
    }
}

public class MyTool {
    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(RunCommand.class)
                .args(args)
                .execute();
    }
}
```

This works correctly even with complex multi-line content:

```bash
java -jar mytool.jar -c 'public class Hello {
    public static void main(String... args) {
        System.out.println("Hello");
    }
}' firstarg
```

#### CommandRuntime Pre-Tokenized API

The same pre-tokenized execution is available when using `CommandRuntime` directly via `executeCommand(String commandName, String[] args)`:

```java
CommandRuntime runtime = AeshCommandRuntimeBuilder.builder()
        .commandRegistry(myRegistry)
        .build();

// Pre-tokenized args bypass LineParser — safe for any content
runtime.executeCommand("run", new String[]{"-c", "code with \"quotes\"", "arg1"});
```

#### Building an Executor Without Executing

Use `buildExecutor(String commandName, String[] args)` to parse and populate a command without executing it. This is useful when you need to inspect the parsed command object — for example, to extract option values for external processing:

```java
CommandRuntime runtime = AeshCommandRuntimeBuilder.builder()
        .commandRegistry(myRegistry)
        .build();

// Parse the command and populate fields, but don't execute
Executor<?> executor = runtime.buildExecutor("run", new String[]{"-c", "my code", "arg1"});
Execution<?> execution = executor.getExecutions().get(0);
execution.populateCommand();

// Access the populated command object
RunCommand cmd = (RunCommand) execution.getCommand();
System.out.println("Code: " + cmd.getCode());
System.out.println("Arg: " + cmd.getArg());
```

This is the same pre-tokenized path as `executeCommand`, so arguments with special characters are handled correctly.

### Reading User Input Interactively

Commands can read interactive input from users using `CommandInvocation.getShell().readLine()`:

```java
import org.aesh.AeshRuntimeRunner;
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;

@CommandDefinition(name = "configure", description = "Interactive configuration")
class ConfigureCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'i', hasValue = false, description = "Interactive mode")
    private boolean interactive;
    
    @Option(description = "Configuration value")
    private String value;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
        if (interactive) {
            // Read input from the user interactively
            String name = invocation.getShell().readLine("Enter your name: ");
            String email = invocation.getShell().readLine("Enter your email: ");
            String theme = invocation.getShell().readLine("Choose theme (light/dark): ");
            
            invocation.println("\nConfiguration saved:");
            invocation.println("Name: " + name);
            invocation.println("Email: " + email);
            invocation.println("Theme: " + theme);
        } else {
            invocation.println("Value set to: " + value);
        }
        
        return CommandResult.SUCCESS;
    }
}

public class InteractiveTool {
    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(ConfigureCommand.class)
                .args(args)
                .execute();
    }
}
```

Run interactively:

```bash
java -jar mytool.jar configure -i
```

The program will prompt:
```
Enter your name: John Doe
Enter your email: john@example.com
Choose theme (light/dark): dark

Configuration saved:
Name: John Doe
Email: john@example.com
Theme: dark
```

Or run non-interactively:

```bash
java -jar mytool.jar configure --value "myconfig"
```

Output:
```
Value set to: myconfig
```

#### Reading Sensitive Input

For passwords or sensitive data, use `readLine()` with a masking character:

```java
@Override
public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
    String username = invocation.getShell().readLine("Username: ");
    // Mask password input with '*'
    String password = invocation.getShell().readLine(new Prompt("Password: ", '*'));
    
    invocation.println("Authenticating user: " + username);
    // Authenticate...
    
    return CommandResult.SUCCESS;
}
```

### Complete Example

```java
import org.aesh.AeshRuntimeRunner;
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Argument;
import org.aesh.command.option.Option;

import java.util.List;

@CommandDefinition(name = "calculate", description = "Perform calculations")
class CalculateCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'o', description = "Operation: add, subtract, multiply, divide")
    private String operation = "add";
    
    @Argument(description = "Numbers to calculate")
    private List<Integer> numbers;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        if (numbers == null || numbers.isEmpty()) {
            invocation.println("Error: No numbers provided");
            return CommandResult.FAILURE;
        }
        
        int result = numbers.get(0);
        for (int i = 1; i < numbers.size(); i++) {
            switch (operation) {
                case "add":
                    result += numbers.get(i);
                    break;
                case "subtract":
                    result -= numbers.get(i);
                    break;
                case "multiply":
                    result *= numbers.get(i);
                    break;
                case "divide":
                    if (numbers.get(i) != 0) {
                        result /= numbers.get(i);
                    } else {
                        invocation.println("Error: Division by zero");
                        return CommandResult.FAILURE;
                    }
                    break;
            }
        }
        
        invocation.println("Result: " + result);
        return CommandResult.SUCCESS;
    }
}

public class CalculatorApp {
    public static void main(String[] args) {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(CalculateCommand.class)
                .build();
        
        // Execute calculations
        System.out.println(runner.execute("calculate 10 20 30"));
        // Output: Result: 60
        
        System.out.println(runner.execute("calculate -o multiply 5 4 2"));
        // Output: Result: 40
        
        System.out.println(runner.execute("calculate -o subtract 100 25 10"));
        // Output: Result: 65
    }
}
```

### Using with Testing

`AeshRuntimeRunner` is excellent for testing commands:

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CommandTest {
    
    @Test
    void testGreetCommand() {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(GreetCommand.class)
                .build();
        
        String output = runner.execute("greet --name Alice");
        assertTrue(output.contains("Hello, Alice!"));
    }
    
    @Test
    void testCommandWithDefaultValue() {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(GreetCommand.class)
                .build();
        
        String output = runner.execute("greet");
        assertTrue(output.contains("Hello, World!"));
    }
}
```

## Choosing Between Runners

### Use AeshConsoleRunner when:
- Building an interactive CLI application
- Users need to type commands repeatedly
- You want features like history, tab completion, and line editing
- Building a shell or REPL environment

### Use AeshRuntimeRunner when:
- Executing commands programmatically
- Building standalone CLI tools that read from command-line arguments
- Building automation scripts
- Testing commands in unit tests
- Integrating Æsh commands into non-interactive applications
- Processing commands from files or external sources
- Creating tools that can prompt users for input when needed (using `readLine()`)

**Note**: `AeshRuntimeRunner` supports both non-interactive execution (via `.execute()` with command strings) and interactive input (via `CommandInvocation.getShell().readLine()`), making it versatile for various CLI tool scenarios.

## Advanced Configuration

### Custom Command Registry

Both runners support custom command registries:

```java
import org.aesh.command.registry.MutableCommandRegistry;
import org.aesh.command.registry.CommandRegistry;

MutableCommandRegistry registry = new MutableCommandRegistry();
registry.addCommand(new MyCommand());
registry.addCommand(new AnotherCommand());

// For console
AeshConsoleRunner.builder()
    .commandRegistry(registry)
    .start();

// For runtime
AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
    .commandRegistry(registry)
    .build();
```

### Settings Object

The `Settings` object provides fine-grained control:

```java
import org.aesh.terminal.tty.Settings;

Settings settings = Settings.builder()
    // Enable command aliases
    .enableAlias(true)
    
    // Configure history
    .historyFile("/path/to/.myapp_history")
    .historySize(1000)
    
    // Enable export of variables
    .enableExport(true)
    
    // Enable logging
    .logging(true)
    
    // Set mode (EMACS or VI)
    .mode(Settings.Mode.EMACS)
    
    .build();

AeshConsoleRunner.builder()
    .settings(settings)
    .command(MyCommand.class)
    .start();
```

## Error Handling

### Console Runner

In console mode, command errors are displayed to the user:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    try {
        // Command logic
        return CommandResult.SUCCESS;
    } catch (Exception e) {
        invocation.println("Error: " + e.getMessage());
        return CommandResult.FAILURE;
    }
}
```

### Runtime Runner

With runtime execution, handle errors in your application code:

```java
AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
        .command(MyCommand.class)
        .build();

try {
    String result = runner.execute("mycommand --option value");
    if (result.contains("Error")) {
        // Handle command error
    }
} catch (Exception e) {
    // Handle execution exception
    e.printStackTrace();
}
```

## Working Examples

The [aesh-examples repository](https://github.com/aeshell/aesh-examples) contains complete working examples demonstrating both runners:

### Console Runner Examples
- **[getting-started](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started)** - Full console application with multiple commands, history, and aliases
- **[getting-started-input](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started-input)** - Interactive input handling and validation

### Runtime Runner Examples
- **[getting-started-runtime](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started-runtime)** - CLI tool pattern with command-line argument processing
- **[native-runtime](https://github.com/aeshell/aesh-examples/tree/master/aesh/native-runtime)** - GraalVM native image compilation

See the [Examples and Tutorials](../examples) page for detailed information about all available examples.

## Command Lifecycle for Re-Entrant Usage

When a runtime or runner is reused across multiple invocations (e.g., in test suites or long-running applications), Æsh automatically resets option fields to their default values before each parse cycle. However, if your command sets **external state** during execution (static flags, shared configuration, etc.), that state persists between calls.

Implement `CommandLifecycle` to reset external state before each parse:

```java
import org.aesh.command.Command;
import org.aesh.command.CommandDefinition;
import org.aesh.command.CommandLifecycle;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.option.Option;

@CommandDefinition(name = "run", description = "Run with options")
class RunCommand implements Command<CommandInvocation>, CommandLifecycle {

    @Option(hasValue = false, description = "Verbose output")
    private boolean verbose;

    @Option(hasValue = false, description = "Preview mode")
    private boolean preview;

    @Override
    public void beforeParse() {
        // Reset any external state from previous invocations
        Config.setVerbose(false);
        Config.setPreview(false);
    }

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Set external state based on parsed options
        Config.setVerbose(verbose);
        Config.setPreview(preview);
        // ... execute command
        return CommandResult.SUCCESS;
    }
}
```

The `beforeParse()` method is called after Æsh clears its internal option values but before the new command line is parsed. This ensures a clean state for every invocation, whether using `executeCommand()`, `buildExecutor()`, or `AeshRuntimeRunner.execute()`.

## Best Practices

1. **Always add exit command**: For console applications, always call `.addExitCommand()` to allow users to exit gracefully.

2. **Use try-catch in commands**: Handle exceptions within commands and return appropriate `CommandResult` values.

3. **Validate input**: Check for null values and invalid options before processing.

4. **Provide helpful output**: Use `invocation.println()` to give users feedback about command execution.

5. **Configure settings appropriately**: Enable history and aliases for better user experience in console mode.

6. **Test with RuntimeRunner**: Use `AeshRuntimeRunner` in your test suite to verify command behavior.

7. **Register commands before starting**: Ensure all commands are registered before calling `.start()` or `.build()`.
