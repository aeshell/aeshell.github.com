---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'CommandInvocation API'
weight: 14
---

The `CommandInvocation` interface is the primary way commands interact with the shell environment. It provides methods for output, input, shell control, and access to command metadata.

## Overview

When a command is executed, Æsh passes a `CommandInvocation` object to the `execute()` method. This object serves as the bridge between your command logic and the shell environment.

```java
@CommandDefinition(name = "example", description = "Example command")
public class ExampleCommand implements Command<CommandInvocation> {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Use invocation to interact with the shell
        invocation.println("Hello from the command!");
        return CommandResult.SUCCESS;
    }
}
```

## Interface Definition

```java
public interface CommandInvocation {

    // Output methods
    void print(String message);
    void println(String message);

    // Shell access
    Shell getShell();

    // Control methods
    void stop();

    // Help system
    String getHelpInfo();
    String getHelpInfo(String commandName);

    // Configuration access
    CommandInvocationConfiguration getConfiguration();

    // Operator support
    Operator getOperator();

    // Standard input (piped or redirected)
    InputStream getStdin();
    boolean hasStdin();

    // Sub-command mode
    boolean enterSubCommandMode(Command<?> command);
    void exitSubCommandMode();
    boolean isInSubCommandMode();
    CommandContext getCommandContext();
    <T> T getParentValue(String name, Class<T> type);
    <T> T getParentValue(String name, Class<T> type, T defaultValue);
    <T> T getInheritedValue(String name, Class<T> type);
    <T> T getInheritedValue(String name, Class<T> type, T defaultValue);
}
```

## Output Methods

### print(String message)

Outputs text to the console without a trailing newline.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    invocation.print("Processing");
    invocation.print("...");
    invocation.print(" done!\n");
    return CommandResult.SUCCESS;
}
```

**Output:** `Processing... done!`

### println(String message)

Outputs text to the console with a trailing newline.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    invocation.println("Line 1");
    invocation.println("Line 2");
    invocation.println("Line 3");
    return CommandResult.SUCCESS;
}
```

**Output:**
```
Line 1
Line 2
Line 3
```

### Formatted Output

Combine with `String.format()` for formatted output:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    String name = "Alice";
    int count = 42;
    invocation.println(String.format("User: %s, Items: %d", name, count));
    return CommandResult.SUCCESS;
}
```

## Shell Access

### getShell()

Returns the `Shell` object for advanced terminal interaction.

```java
Shell shell = invocation.getShell();
```

The `Shell` interface provides:

| Method | Description |
|--------|-------------|
| `readLine()` | Read a line of input from the user |
| `readLine(String prompt)` | Read input with a custom prompt |
| `readLine(Prompt prompt)` | Read input with a Prompt object (supports masking) |
| `write(String text)` | Write text directly to the terminal |
| `writeln(String text)` | Write text with newline |
| `clear()` | Clear the terminal screen |
| `getSize()` | Get terminal dimensions (rows, columns) |
| `enableAlternateBuffer()` | Switch to alternate screen buffer |
| `enableMainBuffer()` | Switch back to main screen buffer |

### Reading User Input

```java
@Override
public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
    Shell shell = invocation.getShell();
    
    // Simple input
    String name = shell.readLine("Enter your name: ");
    invocation.println("Hello, " + name + "!");
    
    return CommandResult.SUCCESS;
}
```

### Reading Password (Masked Input)

```java
@Override
public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
    Shell shell = invocation.getShell();
    
    String username = shell.readLine("Username: ");
    String password = shell.readLine(new Prompt("Password: ", '*'));
    
    // Authenticate user...
    invocation.println("Authenticating " + username + "...");
    
    return CommandResult.SUCCESS;
}
```

### Getting Terminal Size

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    Shell shell = invocation.getShell();
    Size size = shell.getSize();
    
    invocation.println("Terminal: " + size.getWidth() + "x" + size.getHeight());
    return CommandResult.SUCCESS;
}
```

### Clearing the Screen

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    Shell shell = invocation.getShell();
    shell.clear();
    invocation.println("Screen cleared!");
    return CommandResult.SUCCESS;
}
```

## Control Methods

### stop()

Stops the console/shell. This is typically used when a command needs to terminate the entire application.

```java
@CommandDefinition(name = "exit", description = "Exit the shell")
public class ExitCommand implements Command<CommandInvocation> {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Goodbye!");
        invocation.stop();
        return CommandResult.SUCCESS;
    }
}
```

**Note:** For most applications, using the built-in exit command via `.addExitCommand()` is recommended.

## Help System

### getHelpInfo()

Returns the help text for the current command.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    if (showHelp) {
        invocation.println(invocation.getHelpInfo());
        return CommandResult.SUCCESS;
    }
    // Normal execution...
    return CommandResult.SUCCESS;
}
```

### getHelpInfo(String commandName)

Returns help text for a specific command.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    // Display help for another command
    String copyHelp = invocation.getHelpInfo("copy");
    invocation.println(copyHelp);
    return CommandResult.SUCCESS;
}
```

## Configuration Access

### getConfiguration()

Returns the `CommandInvocationConfiguration` for accessing runtime settings.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    CommandInvocationConfiguration config = invocation.getConfiguration();
    
    // Access configuration values
    // ...
    
    return CommandResult.SUCCESS;
}
```

## Operator Support

### getOperator()

Returns the current command-line operator, useful for piping and chaining commands.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    Operator operator = invocation.getOperator();
    
    switch (operator) {
        case PIPE:
            // Output is being piped to another command
            break;
        case REDIRECT_OUT:
            // Output is being redirected to a file
            break;
        case NONE:
            // Normal execution
            break;
    }
    
    return CommandResult.SUCCESS;
}
```

**Operator values:**

| Operator | Symbol | Description |
|----------|--------|-------------|
| `NONE` | - | No operator, normal execution |
| `PIPE` | `\|` | Output piped to next command |
| `REDIRECT_OUT` | `>` | Output redirected to file |
| `REDIRECT_OUT_APPEND` | `>>` | Output appended to file |
| `REDIRECT_IN` | `<` | Input from file |
| `AND` | `&&` | Execute next if success |
| `OR` | `\|\|` | Execute next if failure |
| `END` | `;` | Execute next unconditionally |

## Standard Input (Pipes and Redirects)

### getStdin()

Returns an `InputStream` when the command is receiving piped or redirected input. Returns `null` if no input is available. Both `cmd1 | cmd2` and `cmd2 < file` deliver input through this method.

```java
@CommandDefinition(name = "uppercase", description = "Convert input to uppercase")
public class UppercaseCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) throws CommandException {
        InputStream stdin = invocation.getStdin();
        if (stdin != null) {
            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(stdin))) {
                String line;
                while ((line = reader.readLine()) != null) {
                    invocation.println(line.toUpperCase());
                }
            } catch (IOException e) {
                throw new CommandException(e);
            }
        }
        return CommandResult.SUCCESS;
    }
}
```

**Usage:** `echo "hello world" | uppercase` or `uppercase < input.txt`

**Output:** `HELLO WORLD`

### hasStdin()

Lightweight check for whether piped or redirected input is available, without allocating any streams.

```java
if (invocation.hasStdin()) {
    // Process piped/redirected input
} else {
    // Interactive mode
}
```

## Sub-Command Mode Methods

These methods support [Sub-Command Mode](/docs/aesh/sub-command-mode), allowing group commands to create interactive contexts where subcommands can access parent values.

### enterSubCommandMode(Command)

Enters sub-command mode for the current group command. Returns `true` if successful, `false` if sub-command mode is disabled.

```java
@GroupCommandDefinition(name = "project", groupCommands = {BuildCommand.class, TestCommand.class})
public class ProjectCommand implements Command<CommandInvocation> {

    @Option(name = "name", required = true)
    private String projectName;

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Project: " + projectName);

        // Enter sub-command mode - prompt changes to "project[myapp]>"
        if (invocation.enterSubCommandMode(this)) {
            invocation.println("Available commands: build, test");
        }

        return CommandResult.SUCCESS;
    }
}
```

### exitSubCommandMode()

Exits the current sub-command mode level. If nested, returns to the parent context. If at the top level, returns to the main prompt.

```java
@CommandDefinition(name = "done", description = "Exit project mode")
public class DoneCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.exitSubCommandMode();
        return CommandResult.SUCCESS;
    }
}
```

### isInSubCommandMode()

Returns `true` if currently in sub-command mode.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    if (invocation.isInSubCommandMode()) {
        invocation.println("Currently in sub-command mode");
    } else {
        invocation.println("Running at top level");
    }
    return CommandResult.SUCCESS;
}
```

### getParentValue(String name, Class&lt;T&gt; type)

Retrieves a value from the parent command by field name. Returns `null` if not found or not in sub-command mode.

```java
@CommandDefinition(name = "build", description = "Build the project")
public class BuildCommand implements Command<CommandInvocation> {

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Get parent's projectName field value
        String projectName = invocation.getParentValue("projectName", String.class);

        if (projectName != null) {
            invocation.println("Building " + projectName);
        } else {
            invocation.println("Error: No project name available");
        }

        return CommandResult.SUCCESS;
    }
}
```

### getParentValue(String name, Class&lt;T&gt; type, T defaultValue)

Same as above, but returns a default value if the parent value is not found.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    // Get parent value with default
    Boolean verbose = invocation.getParentValue("verbose", Boolean.class, false);

    if (verbose) {
        invocation.println("[VERBOSE] Starting build process...");
    }

    return CommandResult.SUCCESS;
}
```

### getInheritedValue(String name, Class&lt;T&gt; type)

Retrieves a value from inherited options (options marked with `inherited = true` on the parent). Returns `null` if not found.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    // Get inherited option value
    Boolean verbose = invocation.getInheritedValue("verbose", Boolean.class);

    if (verbose != null && verbose) {
        invocation.println("[VERBOSE] Detailed output enabled");
    }

    return CommandResult.SUCCESS;
}
```

### getInheritedValue(String name, Class&lt;T&gt; type, T defaultValue)

Same as above, but returns a default value if the inherited value is not found.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    // Get inherited value with default
    String logLevel = invocation.getInheritedValue("logLevel", String.class, "INFO");
    invocation.println("Log level: " + logLevel);

    return CommandResult.SUCCESS;
}
```

### getCommandContext()

Returns the `CommandContext` object for advanced context manipulation. Most use cases are covered by the methods above.

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    CommandContext context = invocation.getCommandContext();

    // Check context depth (nesting level)
    int depth = context.depth();
    invocation.println("Context depth: " + depth);

    // Get the context path (e.g., "module:project")
    String path = context.getContextPath();
    invocation.println("Context path: " + path);

    return CommandResult.SUCCESS;
}
```

### Sub-Command Mode Example

Here's a complete example showing how subcommands access parent values:

```java
@GroupCommandDefinition(
    name = "project",
    description = "Project management",
    groupCommands = {BuildCommand.class, StatusCommand.class}
)
public class ProjectCommand implements Command<CommandInvocation> {

    @Option(name = "name", required = true)
    private String projectName;

    @Option(name = "verbose", hasValue = false, inherited = true)
    private boolean verbose;

    public String getProjectName() { return projectName; }
    public boolean isVerbose() { return verbose; }

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Project: " + projectName);
        invocation.enterSubCommandMode(this);
        return CommandResult.SUCCESS;
    }
}

@CommandDefinition(name = "build", description = "Build the project")
public class BuildCommand implements Command<CommandInvocation> {

    @ParentCommand
    private ProjectCommand parent;  // Direct injection

    @Option(name = "verbose", hasValue = false)
    private boolean verbose;  // Auto-populated from parent's inherited option

    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Method 1: Use @ParentCommand annotation
        String name = parent.getProjectName();

        // Method 2: Use getParentValue()
        String name2 = invocation.getParentValue("projectName", String.class);

        // Method 3: Check inherited values
        Boolean inheritedVerbose = invocation.getInheritedValue("verbose", Boolean.class);

        invocation.println("Building " + name);
        if (verbose) {
            invocation.println("[VERBOSE] Compiling sources...");
        }

        return CommandResult.SUCCESS;
    }
}
```

**Usage:**
```
[myapp]$ project --name=myapp --verbose
Project: myapp
Entering project mode.

project[myapp]> build
Building myapp
[VERBOSE] Compiling sources...

project[myapp]> exit
[myapp]$
```

See [Sub-Command Mode](/docs/aesh/sub-command-mode) for complete documentation.

## Complete Example

```java
@CommandDefinition(
    name = "wizard",
    description = "Interactive setup wizard",
    generateHelp = true
)
public class WizardCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 's', hasValue = false, description = "Skip confirmation")
    private boolean skipConfirmation;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) throws InterruptedException {
        Shell shell = invocation.getShell();
        
        // Clear screen and show header
        shell.clear();
        invocation.println("=== Setup Wizard ===\n");
        
        // Collect user information
        String name = shell.readLine("Enter your name: ");
        String email = shell.readLine("Enter your email: ");
        String password = shell.readLine(new Prompt("Enter password: ", '*'));
        
        // Display terminal info
        Size size = shell.getSize();
        invocation.println("\nTerminal size: " + size.getWidth() + "x" + size.getHeight());
        
        // Confirm
        if (!skipConfirmation) {
            invocation.println("\nConfiguration summary:");
            invocation.println(String.format("  Name: %s", name));
            invocation.println(String.format("  Email: %s", email));
            
            String confirm = shell.readLine("\nSave configuration? (yes/no): ");
            if (!confirm.equalsIgnoreCase("yes")) {
                invocation.println("Setup cancelled.");
                return CommandResult.FAILURE;
            }
        }
        
        // Save configuration...
        invocation.println("\nConfiguration saved successfully!");
        
        return CommandResult.SUCCESS;
    }
}
```

## Custom CommandInvocation

For advanced use cases like dependency injection, you can create a custom `CommandInvocation` subtype that provides access to application services.

### Step 1: Define the Custom Interface

```java
public interface MyCommandInvocation extends CommandInvocation {
    
    // Add custom methods for your application services
    DatabaseConnection getDatabase();
    UserSession getCurrentUser();
    ConfigurationService getConfig();
}
```

### Step 2: Implement the Custom CommandInvocation

```java
import org.aesh.command.invocation.CommandInvocation;
import org.aesh.command.invocation.CommandInvocationConfiguration;
import org.aesh.command.shell.Shell;
import org.aesh.command.Operator;

public class MyCommandInvocationImpl implements MyCommandInvocation {
    
    private final CommandInvocation delegate;
    private final DatabaseConnection database;
    private final UserSession userSession;
    private final ConfigurationService config;
    
    public MyCommandInvocationImpl(
            CommandInvocation delegate,
            DatabaseConnection database,
            UserSession userSession,
            ConfigurationService config) {
        this.delegate = delegate;
        this.database = database;
        this.userSession = userSession;
        this.config = config;
    }
    
    // Delegate standard CommandInvocation methods
    @Override
    public void print(String msg) {
        delegate.print(msg);
    }
    
    @Override
    public void println(String msg) {
        delegate.println(msg);
    }
    
    @Override
    public Shell getShell() {
        return delegate.getShell();
    }
    
    @Override
    public void stop() {
        delegate.stop();
    }
    
    @Override
    public String getHelpInfo() {
        return delegate.getHelpInfo();
    }
    
    @Override
    public String getHelpInfo(String commandName) {
        return delegate.getHelpInfo(commandName);
    }
    
    @Override
    public CommandInvocationConfiguration getConfiguration() {
        return delegate.getConfiguration();
    }
    
    @Override
    public Operator getOperator() {
        return delegate.getOperator();
    }
    
    @Override
    public InputStream getStdin() {
        return delegate.getStdin();
    }
    
    // Custom methods for application services
    @Override
    public DatabaseConnection getDatabase() {
        return database;
    }
    
    @Override
    public UserSession getCurrentUser() {
        return userSession;
    }
    
    @Override
    public ConfigurationService getConfig() {
        return config;
    }
}
```

### Step 3: Create the CommandInvocationProvider

```java
import org.aesh.command.invocation.CommandInvocationProvider;

public class MyCommandInvocationProvider 
        implements CommandInvocationProvider<MyCommandInvocation> {
    
    private final DatabaseConnection database;
    private final UserSession userSession;
    private final ConfigurationService config;
    
    public MyCommandInvocationProvider(
            DatabaseConnection database,
            UserSession userSession,
            ConfigurationService config) {
        this.database = database;
        this.userSession = userSession;
        this.config = config;
    }
    
    @Override
    public MyCommandInvocation enhanceCommandInvocation(
            CommandInvocation commandInvocation) {
        return new MyCommandInvocationImpl(
                commandInvocation,
                database,
                userSession,
                config
        );
    }
}
```

### Step 4: Create Commands Using the Custom Invocation

```java
@CommandDefinition(name = "query", description = "Query database")
public class QueryCommand implements Command<MyCommandInvocation> {
    
    @Argument(description = "SQL query to execute")
    private String query;
    
    @Override
    public CommandResult execute(MyCommandInvocation invocation) {
        // Access injected dependencies
        DatabaseConnection db = invocation.getDatabase();
        UserSession user = invocation.getCurrentUser();
        
        // Check permissions
        if (!user.hasPermission("query.execute")) {
            invocation.println("Permission denied: You cannot execute queries.");
            return CommandResult.FAILURE;
        }
        
        // Execute query
        try {
            invocation.println("Executing query as user: " + user.getUsername());
            ResultSet results = db.executeQuery(query);
            printResults(results, invocation);
            return CommandResult.SUCCESS;
        } catch (SQLException e) {
            invocation.println("Query error: " + e.getMessage());
            return CommandResult.FAILURE;
        }
    }
    
    private void printResults(ResultSet results, MyCommandInvocation invocation) {
        // Print query results...
    }
}

@CommandDefinition(name = "whoami", description = "Show current user")
public class WhoamiCommand implements Command<MyCommandInvocation> {
    
    @Override
    public CommandResult execute(MyCommandInvocation invocation) {
        UserSession user = invocation.getCurrentUser();
        invocation.println("Logged in as: " + user.getUsername());
        invocation.println("Roles: " + String.join(", ", user.getRoles()));
        return CommandResult.SUCCESS;
    }
}
```

### Step 5: Configure and Start Æsh

```java
import org.aesh.command.CommandRuntime;
import org.aesh.command.impl.registry.AeshCommandRegistryBuilder;
import org.aesh.command.registry.CommandRegistry;
import org.aesh.command.settings.Settings;
import org.aesh.command.settings.SettingsBuilder;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineConsole;

public class MyApplication {
    
    public static void main(String[] args) throws Exception {
        // Initialize your application services
        DatabaseConnection database = new DatabaseConnection("jdbc:postgresql://localhost/mydb");
        UserSession userSession = new UserSession("admin", Arrays.asList("admin", "user"));
        ConfigurationService config = new ConfigurationService();
        
        // Create the custom invocation provider
        MyCommandInvocationProvider invocationProvider = new MyCommandInvocationProvider(
                database,
                userSession,
                config
        );
        
        // Build the command registry with your commands
        CommandRegistry<MyCommandInvocation> registry = 
                AeshCommandRegistryBuilder.<MyCommandInvocation>builder()
                        .command(QueryCommand.class)
                        .command(WhoamiCommand.class)
                        .create();
        
        // Configure settings with the custom invocation provider
        Settings<MyCommandInvocation> settings = 
                SettingsBuilder.<MyCommandInvocation>builder()
                        .commandRegistry(registry)
                        .commandInvocationProvider(invocationProvider)
                        .enableHistory(true)
                        .persistHistory(true)
                        .historyFile(new File(System.getProperty("user.home"), ".myapp_history"))
                        .build();
        
        // Create and start the console
        ReadlineConsole console = new ReadlineConsole(settings);
        console.setPrompt("[myapp]$ ");
        console.start();
    }
}
```

### Alternative: Using AeshConsoleRunner (Simplified)

For simpler setups, you can also configure the provider through `SettingsBuilder`:

```java
public class MyApplication {
    
    public static void main(String[] args) {
        // Initialize services
        DatabaseConnection database = new DatabaseConnection("jdbc:postgresql://localhost/mydb");
        UserSession userSession = new UserSession("admin", Arrays.asList("admin", "user"));
        ConfigurationService config = new ConfigurationService();
        
        // Create provider
        MyCommandInvocationProvider provider = new MyCommandInvocationProvider(
                database, userSession, config
        );
        
        // Build settings with the provider
        Settings settings = SettingsBuilder.builder()
                .commandInvocationProvider(provider)
                .build();
        
        // Start the console
        AeshConsoleRunner.builder()
                .settings(settings)
                .command(QueryCommand.class)
                .command(WhoamiCommand.class)
                .addExitCommand()
                .prompt("[myapp]$ ")
                .start();
    }
}
```

### Example Session

```
[myapp]$ whoami
Logged in as: admin
Roles: admin, user

[myapp]$ query SELECT * FROM users LIMIT 5
Executing query as user: admin
+----+----------+-------------------+
| id | username | email             |
+----+----------+-------------------+
| 1  | alice    | alice@example.com |
| 2  | bob      | bob@example.com   |
+----+----------+-------------------+

[myapp]$ exit
Goodbye!
```

See [Advanced Topics - Custom Command Invocation](/docs/aesh/advanced-topics#custom-command-invocation) for more patterns including Spring integration.

## Thread Safety

The `CommandInvocation` object is **not thread-safe**. It should only be used within the `execute()` method on the thread that called it. Do not share the invocation object between threads.

## Best Practices

1. **Always use invocation for output** - Use `invocation.println()` instead of `System.out.println()` to ensure output goes to the correct terminal.

2. **Handle InterruptedException** - Methods like `readLine()` throw `InterruptedException`. Always declare it in your method signature.

3. **Check for null stdin** - `getStdin()` returns `null` when there's no piped or redirected input. Use `hasStdin()` for a lightweight check.

4. **Use Prompt for sensitive input** - Always use `Prompt` with a mask character for passwords and other sensitive data.

5. **Clear sensitive data** - Zero out password character arrays after use.

```java
char[] password = shell.readLine(new Prompt("Password: ", '*')).toCharArray();
// Use password...
Arrays.fill(password, '\0'); // Clear sensitive data
```
