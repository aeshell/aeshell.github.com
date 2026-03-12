---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Advanced Topics'
weight: 17
---

This section covers advanced Æsh features including piping, redirection, aliases, testing, GraalVM native images, and custom command invocation providers.

## Piping and Redirection

Æsh supports Unix-style command operators for piping output between commands, redirecting to files, and conditional execution (`&&`, `||`, `;`).

See [Operators](/docs/aesh/operators) for full documentation including all operator types, examples, and how to write operator-aware commands.

## Command Aliases

Aliases allow users to create shortcuts for frequently used commands.

### Enabling Aliases

```java
Settings settings = SettingsBuilder.builder()
        .enableAlias(true)
        .aliasFile(new File(System.getProperty("user.home"), ".myapp_aliases"))
        .persistAlias(true)
        .build();
```

### Built-in Alias Commands

When aliases are enabled, the following commands become available:

```bash
# Create an alias
alias ll='ls -l'

# List all aliases
alias

# Remove an alias
unalias ll
```

### Adding the Alias Command

Use the extensions library to add alias support:

```java
import org.aesh.extensions.common.AeshAlias;

AeshConsoleRunner.builder()
        .command(MyCommand.class)
        .command(AeshAlias.class)
        .settings(SettingsBuilder.builder()
                .enableAlias(true)
                .build())
        .start();
```

### Programmatic Alias Management

```java
// Access alias manager through settings
AliasManager aliasManager = settings.getAliasManager();

// Add alias
aliasManager.addAlias("ll", "ls -l");

// Get alias value
String value = aliasManager.getAlias("ll");

// Remove alias
aliasManager.removeAlias("ll");

// List all aliases
Map<String, String> aliases = aliasManager.getAliases();
```

## Testing Commands

Æsh provides excellent support for testing commands using `AeshRuntimeRunner`.

### Basic Command Testing

```java
import org.aesh.AeshRuntimeRunner;
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
    
    @Test
    void testRequiredOptionMissing() {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(RequiredOptionCommand.class)
                .build();
        
        String output = runner.execute("mycommand");
        assertTrue(output.contains("required"));
    }
}
```

### Testing with Multiple Commands

```java
@Test
void testCommandInteraction() {
    AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
            .command(SetCommand.class)
            .command(GetCommand.class)
            .build();
    
    runner.execute("set key1 value1");
    String output = runner.execute("get key1");
    
    assertEquals("value1", output.trim());
}
```

### Testing Error Handling

```java
@Test
void testInvalidInput() {
    AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
            .command(CalculateCommand.class)
            .build();
    
    String output = runner.execute("calculate --operation divide 10 0");
    assertTrue(output.contains("Error: Division by zero"));
}
```

### Testing with Custom Configuration

```java
@Test
void testWithCustomSettings() {
    Settings settings = SettingsBuilder.builder()
            .enableAlias(true)
            .build();
    
    AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
            .command(MyCommand.class)
            .settings(settings)
            .build();
    
    // Test with aliases enabled
    String output = runner.execute("mycommand");
    // ...
}
```

### Parameterized Tests

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class ParameterizedCommandTest {
    
    @ParameterizedTest
    @CsvSource({
            "add, 5 3, 8",
            "subtract, 10 4, 6",
            "multiply, 6 7, 42",
            "divide, 20 4, 5"
    })
    void testCalculatorOperations(String operation, String args, String expected) {
        AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
                .command(CalculateCommand.class)
                .build();
        
        String output = runner.execute("calculate -o " + operation + " " + args);
        assertTrue(output.contains(expected));
    }
}
```

## GraalVM Native Image

Æsh applications can be compiled to native executables using GraalVM, providing fast startup and low memory footprint.

> **Tip:** Adding the [Annotation Processor](../annotation-processor) (`aesh-processor`) significantly simplifies GraalVM native images. The generated metadata uses direct `new` calls instead of reflection, reducing the amount of reflection configuration you need to maintain.

### Adding GraalVM Support

Add the GraalVM native-image Maven plugin:

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>0.10.1</version>
    <executions>
        <execution>
            <id>build-native</id>
            <goals>
                <goal>compile-no-fork</goal>
            </goals>
            <phase>package</phase>
        </execution>
    </executions>
    <configuration>
        <mainClass>com.example.MyApp</mainClass>
        <imageName>myapp</imageName>
        <buildArgs>
            <buildArg>--no-fallback</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```

### Reflection Configuration

Æsh uses reflection for command discovery by default. If you are using the [Annotation Processor](../annotation-processor), most of this reflection is eliminated and you may need significantly fewer entries here (primarily just for field injection, which still uses reflection).

Without the annotation processor, create a reflection configuration file at `src/main/resources/META-INF/native-image/reflect-config.json`:

```json
[
    {
        "name": "com.example.MyCommand",
        "allDeclaredConstructors": true,
        "allDeclaredFields": true,
        "allDeclaredMethods": true
    },
    {
        "name": "com.example.AnotherCommand",
        "allDeclaredConstructors": true,
        "allDeclaredFields": true,
        "allDeclaredMethods": true
    }
]
```

### Resource Configuration

For resources like history or alias files:

```json
{
    "resources": {
        "includes": [
            {"pattern": ".*\\.properties$"},
            {"pattern": "META-INF/.*"}
        ]
    }
}
```

### Building Native Image

```bash
# With Maven
mvn package -Pnative

# Or directly with GraalVM
native-image -jar target/myapp.jar myapp
```

### Runtime-Only Commands

For applications that use `AeshRuntimeRunner`:

```java
public class NativeApp {
    public static void main(String[] args) {
        AeshRuntimeRunner.builder()
                .command(MyCommand.class)
                .args(args)
                .execute();
    }
}
```

This pattern works well with native images as it doesn't require terminal interaction setup.

### Working Examples

See the [native-runtime example](https://github.com/aeshell/aesh-examples/tree/master/aesh/native-runtime) for a complete GraalVM native image setup.

## Custom Command Invocation

For dependency injection and custom context, create a custom `CommandInvocation` implementation.

### Defining Custom CommandInvocation

```java
public interface MyCommandInvocation extends CommandInvocation {
    
    // Add custom methods
    DatabaseConnection getDatabase();
    UserSession getCurrentUser();
    ConfigurationService getConfig();
}
```

### Implementing the Provider

```java
import org.aesh.command.invocation.CommandInvocationProvider;
import org.aesh.command.invocation.CommandInvocationBuilder;

public class MyCommandInvocationProvider 
        implements CommandInvocationProvider<MyCommandInvocation> {
    
    private final DatabaseConnection database;
    private final UserSession session;
    private final ConfigurationService config;
    
    public MyCommandInvocationProvider(
            DatabaseConnection database,
            UserSession session,
            ConfigurationService config) {
        this.database = database;
        this.session = session;
        this.config = config;
    }
    
    @Override
    public MyCommandInvocation enhanceCommandInvocation(
            CommandInvocationBuilder<MyCommandInvocation> builder) {
        return new MyCommandInvocationImpl(
                builder.shell(),
                builder.helpInfo(),
                database,
                session,
                config
        );
    }
}
```

### Command Invocation Implementation

```java
public class MyCommandInvocationImpl implements MyCommandInvocation {
    
    private final Shell shell;
    private final String helpInfo;
    private final DatabaseConnection database;
    private final UserSession session;
    private final ConfigurationService config;
    
    public MyCommandInvocationImpl(
            Shell shell,
            String helpInfo,
            DatabaseConnection database,
            UserSession session,
            ConfigurationService config) {
        this.shell = shell;
        this.helpInfo = helpInfo;
        this.database = database;
        this.session = session;
        this.config = config;
    }
    
    // Implement CommandInvocation methods
    @Override
    public void print(String msg) {
        shell.write(msg);
    }
    
    @Override
    public void println(String msg) {
        shell.writeln(msg);
    }
    
    @Override
    public Shell getShell() {
        return shell;
    }
    
    @Override
    public void stop() {
        // Stop the console
    }
    
    @Override
    public String getHelpInfo() {
        return helpInfo;
    }
    
    // Implement custom methods
    @Override
    public DatabaseConnection getDatabase() {
        return database;
    }
    
    @Override
    public UserSession getCurrentUser() {
        return session;
    }
    
    @Override
    public ConfigurationService getConfig() {
        return config;
    }
}
```

### Using Custom Invocation in Commands

```java
@CommandDefinition(name = "query", description = "Execute database query")
public class QueryCommand implements Command<MyCommandInvocation> {
    
    @Argument(description = "SQL query")
    private String query;
    
    @Override
    public CommandResult execute(MyCommandInvocation invocation) {
        // Access injected dependencies
        DatabaseConnection db = invocation.getDatabase();
        UserSession user = invocation.getCurrentUser();
        
        // Check permissions
        if (!user.hasPermission("query.execute")) {
            invocation.println("Permission denied");
            return CommandResult.FAILURE;
        }
        
        // Execute query
        try {
            ResultSet results = db.executeQuery(query);
            printResults(results, invocation);
            return CommandResult.SUCCESS;
        } catch (SQLException e) {
            invocation.println("Query error: " + e.getMessage());
            return CommandResult.FAILURE;
        }
    }
}
```

### Configuring the Provider

```java
DatabaseConnection db = new DatabaseConnection("jdbc:...");
UserSession session = new UserSession("admin");
ConfigurationService config = new ConfigurationService();

MyCommandInvocationProvider provider = new MyCommandInvocationProvider(db, session, config);

Settings settings = SettingsBuilder.builder()
        .commandInvocationProvider(provider)
        .build();

AeshConsoleRunner.builder()
        .settings(settings)
        .command(QueryCommand.class)
        .start();
```

## Environment Variables

Æsh supports environment variable export and substitution.

### Enabling Exports

```java
Settings settings = SettingsBuilder.builder()
        .enableExport(true)
        .exportFile(new File(System.getProperty("user.home"), ".myapp_exports"))
        .build();
```

### Using Export Command

```bash
# Set an environment variable
export MY_VAR=value

# Use in commands (variable references are expanded)
echo $MY_VAR

# Chain variables
export BASE=/opt/app
export CONFIG=$BASE/config

# List all exports
export

# Remove export
unset MY_VAR
```

### Accessing Exports in Commands

Commands can read exported variables through `AeshContext`:

```java
@Override
public CommandResult execute(CommandInvocation invocation) {
    AeshContext ctx = invocation.getConfiguration().getAeshContext();

    // Read a specific variable
    String dbHost = ctx.exportedVariable("DB_HOST");

    // List all exported variable names
    Set<String> names = ctx.exportedVariableNames();

    return CommandResult.SUCCESS;
}
```

### Listening for Export Changes

The `ExportChangeListener` interface lets your application react when users set or change exported variables. This is useful for reconfiguring services, updating prompts, or logging configuration changes.

```java
import org.aesh.command.export.ExportChangeListener;

ExportChangeListener listener = (name, value) -> {
    System.out.println("Variable changed: " + name + "=" + value);
    // Reconfigure services, update state, etc.
};
```

Register the listener through `SettingsBuilder`:

```java
Settings settings = SettingsBuilder.builder()
        .enableExport(true)
        .exportFile(new File(System.getProperty("user.home"), ".myapp_exports"))
        .exportListener(listener)
        .build();
```

The listener fires whenever `export NAME=value` is executed successfully. It receives the variable name and its resolved value (with any `$VAR` references expanded).

#### Example: Dynamic Configuration

```java
ExportChangeListener configListener = (name, value) -> {
    switch (name) {
        case "LOG_LEVEL":
            Logger.getGlobal().setLevel(Level.parse(value));
            break;
        case "OUTPUT_FORMAT":
            outputFormatter.setFormat(value);
            break;
    }
};

Settings settings = SettingsBuilder.builder()
        .enableExport(true)
        .exportListener(configListener)
        .build();
```

```bash
# User changes log level at runtime
$ export LOG_LEVEL=FINE
# Listener fires, log level is updated immediately
```

### System Environment Integration

By default, exports are isolated to the Æsh session. To include the host system's environment variables (read-only) alongside session exports:

```java
SettingsBuilder.builder()
        .enableExport(true)
        .exportUsesSystemEnvironment(true)
        .build();
```

With this enabled, `$HOME`, `$PATH`, and other system variables are accessible in addition to user-defined exports.

## Input Validation

### Command-Level Validation

```java
import org.aesh.command.validator.CommandValidator;
import org.aesh.command.validator.CommandValidatorException;

public class CopyCommandValidator implements CommandValidator<CopyCommand, CommandInvocation> {
    
    @Override
    public void validate(CopyCommand command) throws CommandValidatorException {
        if (command.source == null || command.destination == null) {
            throw new CommandValidatorException("Source and destination are required");
        }
        
        if (command.source.equals(command.destination)) {
            throw new CommandValidatorException("Source and destination cannot be the same");
        }
        
        if (!new File(command.source).exists()) {
            throw new CommandValidatorException("Source file does not exist: " + command.source);
        }
    }
}

@CommandDefinition(
    name = "copy",
    description = "Copy files",
    validator = CopyCommandValidator.class
)
public class CopyCommand implements Command<CommandInvocation> {
    
    @Argument(description = "Source file")
    String source;
    
    @Argument(description = "Destination")
    String destination;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Validation has already passed
        // Safe to use source and destination
        return CommandResult.SUCCESS;
    }
}
```

### Option-Level Validation

```java
import org.aesh.command.validator.OptionValidator;
import org.aesh.command.validator.OptionValidatorException;

public class PortValidator implements OptionValidator<Integer> {
    
    @Override
    public void validate(Integer port) throws OptionValidatorException {
        if (port < 1 || port > 65535) {
            throw new OptionValidatorException("Port must be between 1 and 65535");
        }
        
        if (port < 1024) {
            throw new OptionValidatorException("Ports below 1024 require root privileges");
        }
    }
}

@CommandDefinition(name = "serve", description = "Start server")
public class ServeCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'p', validator = PortValidator.class, description = "Port number")
    private int port = 8080;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Starting server on port " + port);
        return CommandResult.SUCCESS;
    }
}
```

## Man Page Generation

Æsh can generate Unix-style man pages from your command definitions.

### Adding Man Page Support

```java
@CommandDefinition(
    name = "myapp",
    description = "My application description",
    generateHelp = true
)
public class MyCommand implements Command<CommandInvocation> {
    
    @Option(
        shortName = 'v',
        hasValue = false,
        description = "Enable verbose output. When enabled, the command will print detailed " +
                      "information about each step of the process."
    )
    private boolean verbose;
    
    @Option(
        shortName = 'o',
        description = "Output file path. The file will be created if it doesn't exist."
    )
    private String output;
    
    @Argument(description = "Input files to process. Multiple files can be specified.")
    private List<File> files;
}
```

### Programmatic Man Page Generation

```java
import org.aesh.command.man.AeshManGenerator;

// Generate man page content
String manPage = AeshManGenerator.generateManPage(MyCommand.class);

// Write to file
Files.writeString(Path.of("myapp.1"), manPage);
```

## Best Practices Summary

1. **Enable piping for composable commands** - Make commands work well together.

2. **Use AeshRuntimeRunner for testing** - Write comprehensive tests for all commands.

3. **Plan for native images early** - Structure code to work with GraalVM restrictions.

4. **Use custom CommandInvocation for DI** - Inject dependencies cleanly.

5. **Validate thoroughly** - Use both option and command validators.

6. **Document commands well** - Detailed descriptions generate better help and man pages.

7. **Support both interactive and non-interactive modes** - Enable automation and scripting.
