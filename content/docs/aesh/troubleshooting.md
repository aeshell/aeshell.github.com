---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Troubleshooting & FAQ'
weight: 18
---

Common issues, solutions, and frequently asked questions for Æsh development.

## Common Issues

### Terminal Issues

#### Input not working in IDE console

**Problem:** When running from an IDE (IntelliJ, Eclipse, VS Code), keyboard input doesn't work correctly.

**Solution:** IDE consoles often don't provide a proper TTY. Options:

1. **Run from real terminal:** Execute your application from a real terminal/shell instead of the IDE console.

2. **Use the test runner:** For development, use `AeshRuntimeRunner` which doesn't need a TTY:

```java
// Development/testing
AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
        .command(MyCommand.class)
        .build();
String output = runner.execute("mycommand --option value");
```

3. **Configure IDE:** Some IDEs support terminal emulation:
   - IntelliJ: Run → Edit Configurations → Check "Emulate terminal in output console"

#### Colors not displaying

**Problem:** ANSI colors appear as escape codes instead of colors.

**Solution:**

1. **Check terminal support:**
```java
Terminal terminal = connection.getTerminal();
if (terminal.getOutputCapability().supportsAnsi()) {
    // Safe to use colors
}
```

2. **Force ANSI on Windows:**
```java
// Windows 10+ supports ANSI in Windows Terminal and ConEmu
System.setProperty("org.jline.terminal.dumb.color", "true");
```

3. **Use theme-aware colors:** See [Terminal Colors](/docs/readline/terminal-colors) for adaptive colors.

#### Arrow keys not working

**Problem:** Pressing arrow keys prints escape sequences instead of moving cursor.

**Solution:** This usually indicates the terminal is not in raw mode. Ensure you're using the Connection properly:

```java
// Correct: Use readline's connection handling
readline.readline(connection, "$ ", input -> {
    // Arrow keys work here
});

// Wrong: Reading from System.in directly
Scanner scanner = new Scanner(System.in);  // Don't do this
```

#### Terminal size is wrong

**Problem:** Application doesn't fit the terminal or wraps incorrectly.

**Solution:**

```java
// Get actual terminal size
Size size = connection.size();
int width = size.getWidth();
int height = size.getHeight();

// Handle resize events
connection.setSizeHandler(newSize -> {
    // Redraw your UI
    redrawInterface(newSize.getWidth(), newSize.getHeight());
});
```

### Command Issues

#### Command not found

**Problem:** Registered commands aren't recognized.

**Checklist:**

1. **Command is registered:**
```java
AeshConsoleRunner.builder()
        .command(MyCommand.class)  // Is this present?
        .start();
```

2. **Annotation is correct:**
```java
@CommandDefinition(name = "mycommand", description = "...")  // Check name
public class MyCommand implements Command<CommandInvocation> {
```

3. **Class implements Command:**
```java
public class MyCommand implements Command<CommandInvocation> {  // Correct interface?
```

4. **Check registry:**
```java
Set<String> commands = registry.getAllCommandNames();
System.out.println("Registered commands: " + commands);
```

#### Option values not being set

**Problem:** Option fields remain null or have default values.

**Checklist:**

1. **Annotation is on field:**
```java
@Option(shortName = 'v')  // Annotation present
private boolean verbose;  // Field to set
```

2. **Option name matches:**
```bash
# If field is named 'verbose', these work:
mycommand --verbose
mycommand -v  # if shortName = 'v'
```

3. **Has value configuration:**
```java
// For boolean flags
@Option(hasValue = false)  // Don't expect a value

// For valued options
@Option(hasValue = true)  // Expect: --option value
```

4. **Check option parsing:**
```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    System.out.println("verbose = " + verbose);  // Debug output
    return CommandResult.SUCCESS;
}
```

#### Required option not being enforced

**Problem:** Command runs even when required option is missing.

**Solution:** Ensure `required = true`:

```java
@Option(required = true, description = "Required file path")
private String filePath;
```

If still not working, check that you're not using `overrideRequired = true` elsewhere.

### History Issues

#### History not persisting

**Problem:** Command history is lost between sessions.

**Solution:** Configure history file:

```java
Settings settings = SettingsBuilder.builder()
        .enableHistory(true)
        .historyFile(new File(System.getProperty("user.home"), ".myapp_history"))
        .historyPersistent(true)
        .build();
```

#### History file not created

**Problem:** History file doesn't exist after running.

**Checklist:**

1. **Directory exists:** The parent directory must exist
2. **Write permissions:** Application needs write access
3. **Application exits gracefully:** History may not save on crash

```java
// Ensure clean shutdown
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // Flush history
}));
```

### Completion Issues

#### Completions not appearing

**Problem:** Pressing Tab doesn't show completions.

**Checklist:**

1. **Completions are provided:**
```java
List<Completion> completions = Arrays.asList(
        new Completion("option1"),
        new Completion("option2")
);

readline.readline(connection, "$ ", input -> { }, completions);
```

2. **Completer is registered:**
```java
@Option(completer = MyCompleter.class)
private String option;
```

3. **Completer implementation is correct:**
```java
public class MyCompleter implements OptionCompleter<CommandInvocation> {
    @Override
    public void complete(CompleterInvocation invocation) {
        String input = invocation.getGivenCompleteValue();
        for (String option : options) {
            if (option.startsWith(input)) {
                invocation.addCompleterValue(option);
            }
        }
    }
}
```

### Build Issues

#### GraalVM native image fails

**Problem:** Native image compilation fails with reflection errors.

**Solution:** Add reflection configuration:

```json
// META-INF/native-image/reflect-config.json
[
    {
        "name": "com.example.MyCommand",
        "allDeclaredConstructors": true,
        "allDeclaredFields": true,
        "allDeclaredMethods": true
    }
]
```

See [Advanced Topics - GraalVM](/docs/aesh/advanced-topics#graalvm-native-image) for complete setup.

#### ClassNotFoundException at runtime

**Problem:** Command classes not found at runtime.

**Checklist:**

1. **Class is in classpath**
2. **Package name is correct**
3. **For modular Java:** Ensure module exports the package

```java
// module-info.java
module myapp {
    requires org.aesh;
    exports com.example.commands;
}
```

## Frequently Asked Questions

### General

#### Q: What's the difference between Æsh and Æsh Readline?

**A:** 
- **Æsh** is a high-level framework for building CLI applications with commands, options, and arguments. Use annotations like `@CommandDefinition` and `@Option`.
- **Æsh Readline** is the low-level library for terminal input handling (line editing, history, completion). Æsh is built on top of Readline.

**Choose Æsh** for CLI tools. **Choose Readline** for custom terminal UIs.

#### Q: Can I use Æsh with Spring Boot?

**A:** Yes! Create your commands as Spring beans and register them:

```java
@Component
@CommandDefinition(name = "greet", description = "Greet someone")
public class GreetCommand implements Command<CommandInvocation> {
    
    @Autowired
    private GreetingService greetingService;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println(greetingService.greet());
        return CommandResult.SUCCESS;
    }
}

@SpringBootApplication
public class MyApp {
    
    @Autowired
    private List<Command<CommandInvocation>> commands;
    
    @PostConstruct
    public void startShell() {
        AeshConsoleRunner.Builder builder = AeshConsoleRunner.builder();
        commands.forEach(builder::command);
        builder.start();
    }
}
```

#### Q: How do I create subcommands like `git remote add`?

**A:** Use group commands. See [Group Commands](/docs/aesh/group-commands):

```java
@CommandDefinition(name = "git", description = "Git commands")
public class GitCommand implements GroupCommand<CommandInvocation> {
    // ...
}

@CommandDefinition(name = "remote", description = "Remote operations")
public class RemoteCommand implements GroupCommand<CommandInvocation> {
    // Add, remove, etc. as subcommands
}
```

#### Q: Can I run commands programmatically?

**A:** Yes, use `AeshRuntimeRunner`:

```java
AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
        .command(MyCommand.class)
        .build();

String output = runner.execute("mycommand --option value");
```

### Options & Arguments

#### Q: How do I accept multiple values for an option?

**A:** Use a List or array:

```java
@Option(shortName = 'f')
private List<String> files;  // --file a.txt --file b.txt
```

Or with `@OptionList`:

```java
@OptionList(shortName = 'f')
private List<String> files;  // --file a.txt,b.txt
```

#### Q: How do I make an option accept values without the `--` prefix?

**A:** Use `@Argument`:

```java
@Argument(description = "Files to process")
private List<File> files;  // command file1.txt file2.txt
```

#### Q: How do I implement password input?

**A:** Use `askIfNotSet` with a masked prompt:

```java
@Option(shortName = 'p', askIfNotSet = true, 
        description = "Password")
private String password;
```

Or manually:

```java
String password = shell.readLine(new Prompt("Password: ", '*'));
```

#### Q: Can I have options with default values from environment variables?

**A:** Use `defaultValue` with environment lookup:

```java
@Option(shortName = 'h', 
        defaultValue = "${env:DATABASE_HOST:localhost}",
        description = "Database host")
private String host;
```

Or in the command:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    if (host == null) {
        host = System.getenv("DATABASE_HOST");
        if (host == null) host = "localhost";
    }
    // ...
}
```

### Testing

#### Q: How do I test commands?

**A:** Use `AeshRuntimeRunner`:

```java
@Test
void testCommand() {
    AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
            .command(MyCommand.class)
            .build();
    
    String output = runner.execute("mycommand --name test");
    
    assertTrue(output.contains("Hello, test!"));
}
```

#### Q: How do I mock dependencies in commands?

**A:** Use constructor injection or a custom `CommandInvocationProvider`:

```java
// Constructor injection
public class MyCommand implements Command<CommandInvocation> {
    private final MyService service;
    
    public MyCommand(MyService service) {
        this.service = service;
    }
}

// In test
MyService mockService = mock(MyService.class);
when(mockService.getData()).thenReturn("test data");

AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
        .command(new MyCommand(mockService))
        .build();
```

### Remote Access

#### Q: How do I add SSH access to my application?

**A:** Add the SSH dependency and create an `SshTerminal`. See [Remote Connectivity](/docs/readline/connectivity#ssh-connectivity).

#### Q: Can I use Æsh commands over SSH?

**A:** Yes! Register commands with `AeshConsoleRunner` and use it with SSH connection:

```java
SshTerminal ssh = SshTerminal.builder()
        .port(2222)
        .keyPair(new File("hostkey.ser"))
        .connectionHandler(connection -> {
            AeshConsoleRunner.builder()
                    .command(MyCommand.class)
                    .connection(connection)
                    .start();
        })
        .build();
```

### Performance

#### Q: How can I improve startup time?

**A:** 

1. **Minimize commands at startup:** Register only essential commands initially
2. **Use GraalVM native image:** Dramatically reduces startup time
3. **Lazy-load resources:** Don't load data until needed

#### Q: How do I handle long-running commands?

**A:** Don't block the readline thread:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    invocation.println("Starting long operation...");
    
    // Run in background
    CompletableFuture.runAsync(() -> {
        // Long operation
        longOperation();
    }).thenRun(() -> {
        invocation.println("Operation complete!");
    });
    
    return CommandResult.SUCCESS;
}
```

## Getting Help

If you can't find a solution here:

1. **Check the examples:** [aesh-examples repository](https://github.com/aeshell/aesh-examples)
2. **Search issues:** [GitHub Issues](https://github.com/aeshell/aesh/issues)
3. **Ask a question:** Create a new issue with the "question" label
4. **Join discussions:** [GitHub Discussions](https://github.com/aeshell/aesh/discussions)

When reporting issues, please include:
- Æsh version
- Java version
- Operating system
- Minimal reproducible example
- Expected vs actual behavior
