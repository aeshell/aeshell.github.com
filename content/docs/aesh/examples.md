---
date: '2026-01-12T12:00:00+01:00'
draft: false
title: 'Examples and Tutorials'
weight: 14
---

Learning by example is one of the best ways to understand Æsh. The [aesh-examples](https://github.com/aeshell/aesh-examples) repository contains working examples that demonstrate various features and use cases.

## Getting Started Examples

### Simple CLI Application

**[getting-started](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started)**

A basic interactive CLI application that demonstrates:
- Creating commands with `@CommandDefinition`
- Using `AeshConsoleRunner` for interactive mode
- Adding options with `@Option`
- Implementing basic command execution

**What you'll learn:**
- How to set up a simple command
- Basic console runner configuration
- Command registration and execution

**Run the example:**
```bash
cd aesh/getting-started
mvn clean compile exec:java
```

### CLI with Input Handling

**[getting-started-input](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started-input)**

Demonstrates how commands can handle different types of input:
- Reading user input within commands
- Handling interactive prompts
- Processing command arguments

**What you'll learn:**
- Using `CommandInvocation` to interact with users
- Reading input from the shell
- Handling both interactive and non-interactive modes

**Run the example:**
```bash
cd aesh/getting-started-input
mvn clean compile exec:java
```

### Runtime Execution

**[getting-started-runtime](https://github.com/aeshell/aesh-examples/tree/master/aesh/getting-started-runtime)**

Shows how to use `AeshRuntimeRunner` for programmatic command execution:
- Executing commands from command-line arguments
- Reading interactive input in runtime mode
- Single-command execution pattern

**Key code snippet:**
```java
AeshRuntimeRunner.builder()
    .command(RuntimeCommand.class)
    .args(args)
    .execute();
```

**What you'll learn:**
- Using `.args()` to pass command-line arguments
- Interactive input with `getShell().readLine()`
- Building CLI tools (not just interactive shells)

**Run the example:**
```bash
cd aesh/getting-started-runtime
mvn clean compile
mvn exec:java -Dexec.args="test --bar hello"
mvn exec:java -Dexec.args="test --input"
```

## Advanced Examples

### Custom Command Invocation

**[extend-context](https://github.com/aeshell/aesh-examples/tree/master/aesh/extend-context)**

Æsh supports creating custom `CommandInvocation` object types that can be used during command execution. This is useful when you need to:
- Share state between commands
- Provide custom context to commands
- Implement application-specific functionality

**What you'll learn:**
- Creating custom `CommandInvocation` implementations
- Extending command context with custom data
- Advanced command registry configuration
- Passing application state to commands

**Use cases:**
- Configuration management
- User session handling
- Application-wide services

### Native Image with GraalVM

**[native-runtime](https://github.com/aeshell/aesh-examples/tree/master/aesh/native-runtime)**

Demonstrates how to create native executables using GraalVM:
- Compiling Æsh applications to native images
- Configuration for reflection and resources
- Performance benefits of native compilation

**What you'll learn:**
- Setting up GraalVM native image build
- Required reflection configuration
- Creating fast-starting CLI tools
- Deployment without JVM

**Benefits:**
- Instant startup time
- Lower memory footprint
- Single executable distribution
- No JVM installation required
## Example Code Snippets

### Basic Command (from getting-started)

```java
@CommandDefinition(name = "hello", description = "Says hello")
public class HelloCommand implements Command<CommandInvocation> {
    
    @Option(shortName = 'n', description = "Name to greet")
    private String name = "World";
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        invocation.println("Hello, " + name + "!");
        return CommandResult.SUCCESS;
    }
}
```

### Runtime with Input (from getting-started-runtime)

```java
@CommandDefinition(name = "test", description = "Runtime example")
public class RuntimeCommand implements Command {
    
    @Option(hasValue = false)
    private boolean input;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) 
            throws InterruptedException {
        if (input) {
            String answer = invocation.getShell()
                .readLine("What is the answer? ");
            invocation.println("The answer is: " + answer);
        }
        return CommandResult.SUCCESS;
    }
}
```

### Custom Context (from extend-context)

```java
// Custom invocation with application context
public class MyCommandInvocation extends AeshCommandInvocation {
    private final ApplicationContext context;
    
    public MyCommandInvocation(ApplicationContext context, 
                              Shell shell, 
                              CommandInvocationConfiguration config) {
        super(shell, config);
        this.context = context;
    }
    
    public ApplicationContext getApplicationContext() {
        return context;
    }
}

// Use in commands
public class MyCommand implements Command<MyCommandInvocation> {
    @Override
    public CommandResult execute(MyCommandInvocation invocation) {
        ApplicationContext ctx = invocation.getApplicationContext();
        // Use shared context...
        return CommandResult.SUCCESS;
    }
}
```

## Additional Resources

- [Æsh GitHub Repository](https://github.com/aeshell/aesh)
- [Æsh Examples Repository](https://github.com/aeshell/aesh-examples)
- [Getting Started Guide](../getting-started)
- [Console and Runtime Runners](../runners)

## Need Help?

If you're stuck or have questions about the examples:
- Check the README in each example directory
- Review the inline code comments
- Open an issue on [GitHub](https://github.com/aeshell/aesh-examples/issues)
- Refer back to the main documentation sections
