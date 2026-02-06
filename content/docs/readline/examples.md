---
date: '2026-01-12T09:00:00+01:00'
draft: false
title: 'Examples and Tutorials'
weight: 11
---

The [aesh-examples](https://github.com/aeshell/aesh-examples) repository provides complete, working examples that demonstrate Æsh Readline features and best practices. These examples are invaluable for learning how to use Readline in real-world scenarios.

## Available Examples

### 1. Getting Started

**Repository:** [readline/getting-started](https://github.com/aeshell/aesh-examples/tree/master/readline/getting-started)

A minimal example showing basic readline functionality.

**What you'll learn:**
- Creating a basic readline instance
- Reading user input
- Handling input events
- Simple prompt configuration

**Perfect for:** First-time users learning the basics

### 2. Getting Started - Extended

**Repository:** [readline/getting-started-ext](https://github.com/aeshell/aesh-examples/tree/master/readline/getting-started-ext)

A more comprehensive example expanding on the basics.

**What you'll learn:**
- Advanced prompt customization
- Multiple input handlers
- Configuration options
- Event handling patterns

**Perfect for:** Users ready to explore beyond basic examples

### 3. Shell Example

**Repository:** [readline/shell](https://github.com/aeshell/aesh-examples/tree/master/readline/shell)

A simple shell implementation demonstrating command execution.

**What you'll learn:**
- Building a basic shell interface
- Command parsing and execution
- Process handling
- Error handling

**Perfect for:** Building command-line applications

### 4. Snake Game

**Repository:** [readline/snake](https://github.com/aeshell/aesh-examples/tree/master/readline/snake)

The classic Snake game implemented using Readline terminal control.

**What you'll learn:**
- Advanced terminal control
- Screen manipulation
- Real-time input handling
- ANSI escape sequences
- Game loop implementation

**Perfect for:** Learning terminal graphics and control sequences

**Key features demonstrated:**
- Cursor positioning
- Screen clearing
- Color output
- Non-blocking input
- Terminal size detection

### 5. SSH Shell

**Repository:** [readline/shell-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-ssh)

The shell example running over SSH protocol.

**What you'll learn:**
- SSH server setup
- Remote terminal connections
- SSH authentication
- Host key management
- Connection handling

**Perfect for:** Building remote shell applications


### 6. Telnet Shell

**Repository:** [readline/shell-telnet](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-telnet)

The shell example running over Telnet protocol.

**What you'll learn:**
- Telnet server configuration
- Legacy protocol support
- Telnet negotiation
- Connection management

**Perfect for:** Supporting legacy systems
``

### 7. WebSocket Shell

**Repository:** [readline/shell-websocket](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-websocket)

The shell example running over WebSocket protocol.

**What you'll learn:**
- WebSocket terminal setup
- Web-based terminal access
- Browser integration
- Modern web connectivity

**Perfect for:** Web-based terminal applications

### 8. Command Mirror SSH

**Repository:** [readline/cmd-mirror-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/cmd-mirror-ssh)

A readline application that mirrors commands over SSH, executing them on the host OS.

**What you'll learn:**
- OS command execution
- Process output capture
- SSH integration
- Real-time output streaming

**Perfect for:** Understanding remote command execution

> **Security Warning:** This example executes user input as OS commands without validation. It is for educational purposes only and should not be used in production without proper command validation and security measures.

### Basic Readline Usage (from getting-started)

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.terminal.Connection;

public class SimpleReadline {
    public static void main(String[] args) {
        // Create readline instance
        Readline readline = ReadlineBuilder.builder().build();
        
        // Create connection (stdin/stdout by default)
        Connection connection = new TerminalConnection();
        
        // Read user input
        readline.readline(connection, "prompt> ", input -> {
            if (input != null && input.equals("exit")) {
                connection.close();
            } else {
                connection.write("You entered: " + input + "\n");
                // Continue reading
                readInput(connection, readline);
            }
        });
    }
}
```

### SSH Terminal Server (from shell-ssh)

```java
import org.aesh.terminal.ssh.SshTerminal;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;

public class SshShell {
    public static void main(String[] args) {
        SshTerminal ssh = SshTerminal.builder()
                .host("0.0.0.0")
                .port(2222)
                .keyPair(new File("hostkey.ser"))
                .connectionHandler(connection -> {
                    Readline readline = ReadlineBuilder.builder().build();
                    startShell(connection, readline);
                })
                .build();
        
        ssh.start();
        System.out.println("SSH server started on port 2222");
    }
    
    private static void startShell(Connection conn, Readline readline) {
        readline.readline(conn, "ssh-shell$ ", input -> {
            if (input != null && input.equals("exit")) {
                conn.write("Goodbye!\n");
                conn.close();
            } else {
                // Handle command
                handleCommand(conn, input);
                // Continue reading
                startShell(conn, readline);
            }
        });
    }
}
```

### Terminal Control (from snake)

```java
import org.aesh.terminal.tty.Size;
import org.aesh.readline.terminal.formatting.TerminalString;
import org.aesh.utils.ANSI;

public class SnakeGame {
    private void drawScreen(Connection connection) {
        // Clear screen
        connection.write(ANSI.CLEAR_SCREEN);
        
        // Move cursor to position
        connection.write(ANSI.CURSOR_START);
        
        // Draw game border
        Size size = connection.size();
        for (int i = 0; i < size.getWidth(); i++) {
            connection.write("─");
        }
        
        // Draw snake with color
        connection.write(ANSI.GREEN_TEXT);
        drawSnake();
        connection.write(ANSI.DEFAULT_TEXT);
        
        // Draw food
        connection.write(ANSI.RED_TEXT);
        connection.write("@");
        connection.write(ANSI.DEFAULT_TEXT);
    }
}
```

## Additional Resources

### Documentation
- [Getting Started Guide](../getting-started) - Basic Readline setup
- [Readline API](../readline-api) - Core API reference
- [Terminal Control](../terminal) - Terminal manipulation
- [Remote Connectivity](../connectivity) - SSH, Telnet, WebSocket setup

### Example Repository
- [GitHub: aesh-examples](https://github.com/aeshell/aesh-examples)
- Each example includes a README with specific instructions
- Full Maven build configuration included

### Community
- [GitHub Issues](https://github.com/aeshell/aesh/issues) - Report issues or ask questions
- [GitHub Discussions](https://github.com/aeshell/aesh/discussions) - Community discussions

## Contributing Examples

If you've built something interesting with Æsh Readline and want to share it:

1. Fork the [aesh-examples](https://github.com/aeshell/aesh-examples) repository
2. Add your example following the existing structure
3. Include a README with setup instructions
4. Submit a pull request

We welcome examples showing:
- Real-world use cases
- Integration with other libraries
- Creative terminal applications
- Performance optimization techniques
- Testing strategies

## Next Steps

1. **Clone the repository:**
   ```bash
   git clone https://github.com/aeshell/aesh-examples.git
   ```

2. **Start with getting-started:**
   ```bash
   cd aesh-examples/readline/getting-started
   mvn clean compile exec:java
   ```

3. **Experiment and modify** - All examples are designed to be modified and extended

4. **Build your own application** - Use the examples as templates for your projects
