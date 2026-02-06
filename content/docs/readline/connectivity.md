---
date: '2026-01-26T10:00:00+01:00'
draft: false
title: 'Remote Connectivity'
weight: 10
---

Æsh Readline supports remote terminal connections via SSH, Telnet, and WebSockets. This allows you to build terminal applications that users can access over the network.

## Overview

Remote connectivity enables:
- **SSH terminals** - Secure, authenticated remote access
- **Telnet terminals** - Simple network terminal access (not recommended for production)
- **WebSocket terminals** - Browser-based terminal access

All remote terminals use the same `Connection` abstraction as local terminals, so your readline code works unchanged.

## Architecture

```
┌─────────────────────────────────────┐
│         Your Application            │
│    (Uses Readline + Connection)     │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│      Connection Interface           │
│   (Same API for all terminals)      │
└──────────────┬──────────────────────┘
               │
      ┌────────┼────────┐
      │        │        │
      ▼        ▼        ▼
┌─────────┐ ┌───────┐ ┌────────────┐
│   SSH   │ │Telnet │ │ WebSocket  │
│ Server  │ │Server │ │   Server   │
└─────────┘ └───────┘ └────────────┘
      │        │        │
      ▼        ▼        ▼
┌─────────┐ ┌───────┐ ┌────────────┐
│SSH Clnt │ │Telnet │ │  Browser   │
│ (PuTTY) │ │Client │ │  (xterm.js)│
└─────────┘ └───────┘ └────────────┘
```

## SSH Connectivity

SSH (Secure Shell) provides encrypted, authenticated terminal access. This is the recommended protocol for production use.

### Maven Dependency

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-terminal-ssh</artifactId>
    <version>2.6</version>
</dependency>
```

### Gradle Dependency

```groovy
implementation 'org.aesh:aesh-terminal-ssh:2.6'
```

### Basic SSH Server

```java
import org.aesh.terminal.ssh.SshTerminal;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.terminal.Connection;
import java.io.File;

public class SshShell {
    
    public static void main(String[] args) throws Exception {
        SshTerminal ssh = SshTerminal.builder()
                .host("0.0.0.0")      // Listen on all interfaces
                .port(2222)            // SSH port
                .keyPair(new File("hostkey.ser"))  // Host key file
                .connectionHandler(SshShell::handleConnection)
                .build();
        
        System.out.println("SSH server started on port 2222");
        System.out.println("Connect with: ssh -p 2222 user@localhost");
        
        ssh.start();
        
        // Keep running
        Thread.currentThread().join();
    }
    
    private static void handleConnection(Connection connection) {
        Readline readline = ReadlineBuilder.builder().build();
        
        connection.write("Welcome to SSH Shell!\n");
        read(connection, readline);
    }
    
    private static void read(Connection connection, Readline readline) {
        readline.readline(connection, "[ssh]$ ", input -> {
            if (input == null || input.equals("exit")) {
                connection.write("Goodbye!\n");
                connection.close();
                return;
            }
            
            connection.write("You entered: " + input + "\n");
            read(connection, readline);
        });
    }
}
```

### SSH Builder Options

| Method | Type | Description |
|--------|------|-------------|
| `host(String)` | `String` | Bind address (default: "0.0.0.0") |
| `port(int)` | `int` | SSH port (default: 22) |
| `keyPair(File)` | `File` | Host key file (generated if not exists) |
| `keyPair(KeyPairProvider)` | `KeyPairProvider` | Custom key pair provider |
| `connectionHandler(Consumer<Connection>)` | `Consumer` | Handler for new connections |
| `passwordAuthenticator(PasswordAuthenticator)` | `PasswordAuthenticator` | Password authentication |
| `publickeyAuthenticator(PublickeyAuthenticator)` | `PublickeyAuthenticator` | Public key authentication |
| `idleTimeout(long)` | `long` | Connection idle timeout (ms) |
| `welcomeBanner(String)` | `String` | Banner shown before login |

### Host Key Generation

SSH requires a host key for secure communication. If the key file doesn't exist, it's generated automatically:

```java
// Auto-generate if not exists
SshTerminal ssh = SshTerminal.builder()
        .keyPair(new File("hostkey.ser"))
        .build();

// Custom key provider
import org.apache.sshd.common.keyprovider.KeyPairProvider;
import org.apache.sshd.server.keyprovider.SimpleGeneratorHostKeyProvider;

KeyPairProvider keyProvider = new SimpleGeneratorHostKeyProvider(
        new File("myapp_hostkey.ser").toPath()
);

SshTerminal ssh = SshTerminal.builder()
        .keyPair(keyProvider)
        .build();
```

### Password Authentication

```java
import org.apache.sshd.server.auth.password.PasswordAuthenticator;

PasswordAuthenticator authenticator = (username, password, session) -> {
    // Validate credentials
    return validateCredentials(username, password);
};

SshTerminal ssh = SshTerminal.builder()
        .host("0.0.0.0")
        .port(2222)
        .keyPair(new File("hostkey.ser"))
        .passwordAuthenticator(authenticator)
        .connectionHandler(this::handleConnection)
        .build();
```

### Public Key Authentication

```java
import org.apache.sshd.server.auth.pubkey.PublickeyAuthenticator;
import java.security.PublicKey;

PublickeyAuthenticator keyAuth = (username, key, session) -> {
    // Check if the public key is authorized for this user
    return authorizedKeys.contains(key);
};

SshTerminal ssh = SshTerminal.builder()
        .publickeyAuthenticator(keyAuth)
        .build();
```

### Combined Authentication

```java
SshTerminal ssh = SshTerminal.builder()
        .host("0.0.0.0")
        .port(2222)
        .keyPair(new File("hostkey.ser"))
        // Allow both password and public key
        .passwordAuthenticator((user, pass, session) -> validatePassword(user, pass))
        .publickeyAuthenticator((user, key, session) -> validatePublicKey(user, key))
        .connectionHandler(this::handleConnection)
        .build();
```

### Complete SSH Example

```java
import org.aesh.terminal.ssh.SshTerminal;
import org.aesh.readline.*;
import org.aesh.readline.history.FileHistory;
import org.aesh.terminal.Connection;
import org.apache.sshd.server.auth.password.PasswordAuthenticator;

import java.io.File;
import java.util.*;

public class SecureShell {
    
    private final Map<String, String> users = new HashMap<>();
    private final Map<String, Consumer<String[]>> commands = new HashMap<>();
    
    public SecureShell() {
        // Setup users
        users.put("admin", "secret123");
        users.put("user", "password");
        
        // Setup commands
        commands.put("help", this::helpCommand);
        commands.put("whoami", this::whoamiCommand);
        commands.put("date", this::dateCommand);
        commands.put("uptime", this::uptimeCommand);
    }
    
    public void start() throws Exception {
        PasswordAuthenticator authenticator = (username, password, session) -> {
            String storedPassword = users.get(username);
            return storedPassword != null && storedPassword.equals(password);
        };
        
        SshTerminal ssh = SshTerminal.builder()
                .host("0.0.0.0")
                .port(2222)
                .keyPair(new File("hostkey.ser"))
                .welcomeBanner("Welcome to SecureShell\n")
                .passwordAuthenticator(authenticator)
                .connectionHandler(this::handleConnection)
                .build();
        
        System.out.println("SSH Server started on port 2222");
        ssh.start();
        
        // Wait for shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutting down...");
            ssh.stop();
        }));
        
        Thread.currentThread().join();
    }
    
    private void handleConnection(Connection connection) {
        String username = connection.getSession().getUsername();
        
        Readline readline = ReadlineBuilder.builder()
                .history(new InMemoryHistory(100))
                .build();
        
        connection.write("\nWelcome, " + username + "!\n");
        connection.write("Type 'help' for available commands.\n\n");
        
        readLoop(connection, readline, username);
    }
    
    private void readLoop(Connection connection, Readline readline, String username) {
        String prompt = username + "@myhost:~$ ";
        
        readline.readline(connection, prompt, input -> {
            if (input == null || input.trim().equalsIgnoreCase("exit")) {
                connection.write("Goodbye, " + username + "!\n");
                connection.close();
                return;
            }
            
            if (!input.trim().isEmpty()) {
                executeCommand(connection, input.trim(), username);
            }
            
            readLoop(connection, readline, username);
        });
    }
    
    private void executeCommand(Connection connection, String input, String username) {
        String[] parts = input.split("\\s+");
        String cmd = parts[0].toLowerCase();
        
        Consumer<String[]> handler = commands.get(cmd);
        if (handler != null) {
            handler.accept(new String[]{connection, username, input});
        } else {
            connection.write("Unknown command: " + cmd + "\n");
        }
    }
    
    // Command implementations...
    
    public static void main(String[] args) throws Exception {
        new SecureShell().start();
    }
}
```

## Telnet Connectivity

Telnet provides simple, unencrypted terminal access. **Use only for development or internal networks.**

### Maven Dependency

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-terminal-telnet</artifactId>
    <version>2.6</version>
</dependency>
```

### Basic Telnet Server

```java
import org.aesh.terminal.telnet.TelnetTerminal;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.terminal.Connection;

public class TelnetShell {
    
    public static void main(String[] args) throws Exception {
        TelnetTerminal telnet = TelnetTerminal.builder()
                .host("0.0.0.0")
                .port(2323)
                .connectionHandler(TelnetShell::handleConnection)
                .build();
        
        System.out.println("Telnet server started on port 2323");
        System.out.println("Connect with: telnet localhost 2323");
        
        telnet.start();
        Thread.currentThread().join();
    }
    
    private static void handleConnection(Connection connection) {
        Readline readline = ReadlineBuilder.builder().build();
        
        connection.write("Welcome to Telnet Shell!\n");
        connection.write("Type 'exit' to disconnect.\n\n");
        
        read(connection, readline);
    }
    
    private static void read(Connection connection, Readline readline) {
        readline.readline(connection, "[telnet]$ ", input -> {
            if (input == null || input.equals("exit")) {
                connection.write("Goodbye!\n");
                connection.close();
                return;
            }
            
            connection.write("Echo: " + input + "\n");
            read(connection, readline);
        });
    }
}
```

### Telnet Builder Options

| Method | Type | Description |
|--------|------|-------------|
| `host(String)` | `String` | Bind address (default: "0.0.0.0") |
| `port(int)` | `int` | Telnet port (default: 23) |
| `connectionHandler(Consumer<Connection>)` | `Consumer` | Handler for new connections |
| `idleTimeout(long)` | `long` | Connection idle timeout (ms) |

## HTTP/WebSocket Connectivity

WebSocket terminals enable browser-based terminal access, perfect for web applications.

### Maven Dependency

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-terminal-http</artifactId>
    <version>2.6</version>
</dependency>
```

### Basic WebSocket Server

```java
import org.aesh.terminal.http.HttpTerminal;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.terminal.Connection;

public class WebTerminal {
    
    public static void main(String[] args) throws Exception {
        HttpTerminal http = HttpTerminal.builder()
                .host("0.0.0.0")
                .port(8080)
                .webSocketPath("/terminal")
                .connectionHandler(WebTerminal::handleConnection)
                .build();
        
        System.out.println("WebSocket terminal started");
        System.out.println("Open browser to: http://localhost:8080/terminal");
        
        http.start();
        Thread.currentThread().join();
    }
    
    private static void handleConnection(Connection connection) {
        Readline readline = ReadlineBuilder.builder().build();
        
        connection.write("Welcome to Web Terminal!\r\n");
        read(connection, readline);
    }
    
    private static void read(Connection connection, Readline readline) {
        readline.readline(connection, "[web]$ ", input -> {
            if (input == null || input.equals("exit")) {
                connection.write("Goodbye!\r\n");
                connection.close();
                return;
            }
            
            connection.write("You entered: " + input + "\r\n");
            read(connection, readline);
        });
    }
}
```

### WebSocket Builder Options

| Method | Type | Description |
|--------|------|-------------|
| `host(String)` | `String` | Bind address (default: "0.0.0.0") |
| `port(int)` | `int` | HTTP port (default: 8080) |
| `webSocketPath(String)` | `String` | WebSocket endpoint path |
| `connectionHandler(Consumer<Connection>)` | `Consumer` | Handler for new connections |
| `staticFilesPath(String)` | `String` | Path to serve static files |
| `idleTimeout(long)` | `long` | Connection idle timeout (ms) |

### Browser Client with xterm.js

Create an HTML client using xterm.js:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Web Terminal</title>
    <link rel="stylesheet" href="https://unpkg.com/xterm/css/xterm.css" />
    <script src="https://unpkg.com/xterm/lib/xterm.js"></script>
    <script src="https://unpkg.com/xterm-addon-attach/lib/xterm-addon-attach.js"></script>
    <script src="https://unpkg.com/xterm-addon-fit/lib/xterm-addon-fit.js"></script>
    <style>
        body { margin: 0; padding: 20px; background: #1e1e1e; }
        #terminal { width: 100%; height: calc(100vh - 40px); }
    </style>
</head>
<body>
    <div id="terminal"></div>
    <script>
        const term = new Terminal({
            cursorBlink: true,
            fontSize: 14,
            fontFamily: 'Menlo, Monaco, "Courier New", monospace'
        });
        
        const fitAddon = new FitAddon.FitAddon();
        term.loadAddon(fitAddon);
        
        term.open(document.getElementById('terminal'));
        fitAddon.fit();
        
        // Connect to WebSocket
        const socket = new WebSocket('ws://localhost:8080/terminal');
        const attachAddon = new AttachAddon.AttachAddon(socket);
        term.loadAddon(attachAddon);
        
        // Handle window resize
        window.addEventListener('resize', () => fitAddon.fit());
        
        // Handle disconnection
        socket.onclose = () => {
            term.write('\r\n\x1b[31mConnection closed\x1b[0m\r\n');
        };
        
        socket.onerror = (error) => {
            term.write('\r\n\x1b[31mConnection error\x1b[0m\r\n');
        };
    </script>
</body>
</html>
```

### Serving Static Files

```java
HttpTerminal http = HttpTerminal.builder()
        .host("0.0.0.0")
        .port(8080)
        .webSocketPath("/terminal")
        .staticFilesPath("src/main/resources/static")  // Serve HTML/JS/CSS
        .connectionHandler(this::handleConnection)
        .build();
```

## Connection Management

### Connection Interface

All remote terminals provide a `Connection` object with the same interface:

```java
public interface Connection {
    
    // Write to terminal
    void write(String text);
    void write(byte[] bytes);
    void write(int[] codepoints);
    
    // Close connection
    void close();
    
    // Terminal size
    Size size();
    void setSizeHandler(Consumer<Size> handler);
    
    // Signal handling
    void setSignalHandler(Consumer<Signal> handler);
    
    // Input handling
    void setStdinHandler(Consumer<int[]> handler);
    
    // Connection state
    boolean isOpen();
    
    // Session information (SSH)
    Session getSession();
}
```

### Handling Terminal Resize

```java
private void handleConnection(Connection connection) {
    // Get initial size
    Size size = connection.size();
    System.out.println("Terminal: " + size.getWidth() + "x" + size.getHeight());
    
    // Handle resize events
    connection.setSizeHandler(newSize -> {
        System.out.println("Resized: " + newSize.getWidth() + "x" + newSize.getHeight());
        // Redraw UI if needed
    });
    
    // Continue with readline...
}
```

### Handling Signals

```java
connection.setSignalHandler(signal -> {
    switch (signal) {
        case INT:   // Ctrl-C
            System.out.println("Interrupted");
            break;
        case QUIT:  // Ctrl-\
            System.out.println("Quit");
            break;
        case TSTP:  // Ctrl-Z
            System.out.println("Suspend");
            break;
    }
});
```

## Running Multiple Servers

Run multiple connection types simultaneously:

```java
public class MultiProtocolServer {
    
    public static void main(String[] args) throws Exception {
        // SSH on port 2222
        SshTerminal ssh = SshTerminal.builder()
                .port(2222)
                .keyPair(new File("hostkey.ser"))
                .passwordAuthenticator((u, p, s) -> authenticate(u, p))
                .connectionHandler(MultiProtocolServer::handleConnection)
                .build();
        
        // Telnet on port 2323 (development only)
        TelnetTerminal telnet = TelnetTerminal.builder()
                .port(2323)
                .connectionHandler(MultiProtocolServer::handleConnection)
                .build();
        
        // WebSocket on port 8080
        HttpTerminal http = HttpTerminal.builder()
                .port(8080)
                .webSocketPath("/terminal")
                .connectionHandler(MultiProtocolServer::handleConnection)
                .build();
        
        // Start all servers
        ssh.start();
        telnet.start();
        http.start();
        
        System.out.println("Servers started:");
        System.out.println("  SSH:       ssh -p 2222 user@localhost");
        System.out.println("  Telnet:    telnet localhost 2323");
        System.out.println("  WebSocket: http://localhost:8080/terminal");
        
        Thread.currentThread().join();
    }
    
    private static void handleConnection(Connection connection) {
        // Same handler for all connection types!
        Readline readline = ReadlineBuilder.builder().build();
        read(connection, readline);
    }
    
    private static void read(Connection connection, Readline readline) {
        readline.readline(connection, "$ ", input -> {
            if (input == null || input.equals("exit")) {
                connection.write("Goodbye!\n");
                connection.close();
                return;
            }
            
            connection.write("Echo: " + input + "\n");
            read(connection, readline);
        });
    }
}
```

## Server Lifecycle

### Starting and Stopping

```java
// Start server
ssh.start();
telnet.start();
http.start();

// Check if running
if (ssh.isRunning()) {
    System.out.println("SSH server is running");
}

// Stop server (closes all connections)
ssh.stop();
telnet.stop();
http.stop();
```

### Graceful Shutdown

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Shutting down servers...");
    
    ssh.stop();
    telnet.stop();
    http.stop();
    
    System.out.println("Servers stopped");
}));
```

## Security Considerations

### SSH Security

1. **Use strong host keys** - Generate RSA 4096 or Ed25519 keys
2. **Prefer public key authentication** - Disable password auth in production
3. **Use non-standard ports** - Avoid port 22 to reduce scanning
4. **Implement rate limiting** - Prevent brute force attacks
5. **Log authentication attempts** - Monitor for suspicious activity

### Telnet Security

**Telnet transmits everything in plaintext. Never use in production!**

- No encryption
- No authentication
- Credentials visible to network sniffers

### WebSocket Security

1. **Use WSS (WebSocket Secure)** - Always use TLS in production
2. **Implement authentication** - Validate users before terminal access
3. **Use CORS** - Restrict which origins can connect
4. **Set appropriate timeouts** - Close idle connections

## Working Examples

The [aesh-examples repository](https://github.com/aeshell/aesh-examples) contains complete working examples:

| Example | Description |
|---------|-------------|
| [shell-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-ssh) | SSH server with authentication |
| [shell-telnet](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-telnet) | Telnet server |
| [shell-websocket](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-websocket) | WebSocket terminal with browser client |
| [cmd-mirror-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/cmd-mirror-ssh) | Command mirroring over SSH |

## Best Practices

1. **Use SSH for production** - It's the only secure option
2. **Always authenticate** - Never allow anonymous access
3. **Handle disconnections gracefully** - Clean up resources
4. **Implement timeouts** - Close idle connections
5. **Log access** - Track who connects and when
6. **Test with real clients** - Verify compatibility

```java
// Production configuration example
SshTerminal ssh = SshTerminal.builder()
        .host("0.0.0.0")
        .port(2222)
        .keyPair(new File("/etc/myapp/hostkey.ser"))
        .idleTimeout(300000)  // 5 minutes
        .publickeyAuthenticator(this::validatePublicKey)
        .connectionHandler(connection -> {
            logConnection(connection);
            handleConnection(connection);
        })
        .build();
```
