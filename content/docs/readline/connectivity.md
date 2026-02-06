---
date: '2026-02-06T10:00:00+01:00'
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

WebSocket terminals enable browser-based terminal access, perfect for web applications. The terminal-http module uses Netty for high-performance WebSocket handling and includes a default web page using [xterm.js](https://github.com/xtermjs/xterm.js).

### Features

The terminal-http module provides:

- **Dynamic terminal resize** - Terminal automatically adjusts when browser window is resized
- **Client capability reporting** - Browser reports terminal type, color depth, and features on connect
- **Modern terminal type** - Uses `xterm-256color` for full color and capability support
- **xterm.js integration** - Professional terminal rendering with cursor blinking, selection, and more

### Maven Dependency

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>terminal-http</artifactId>
    <version>3.0</version>
</dependency>
```

### Basic WebSocket Server

```java
import org.aesh.terminal.http.netty.NettyWebsocketTtyBootstrap;
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;
import org.aesh.terminal.Connection;

import java.util.concurrent.TimeUnit;

public class WebTerminal {

    public static synchronized void main(String[] args) throws Exception {
        NettyWebsocketTtyBootstrap bootstrap = new NettyWebsocketTtyBootstrap()
                .setHost("localhost")
                .setPort(8080);

        bootstrap.start(WebTerminal::handleConnection).get(10, TimeUnit.SECONDS);

        System.out.println("WebSocket terminal started on http://localhost:8080");
        WebTerminal.class.wait();
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

### WebSocket Bootstrap Options

| Method | Type | Description |
|--------|------|-------------|
| `setHost(String)` | `String` | Bind address (default: "localhost") |
| `setPort(int)` | `int` | HTTP port (default: 8080) |
| `setResourcePath(String)` | `String` | Classpath path for static files (default: "/org/aesh/terminal/http") |
| `setServeStaticFiles(boolean)` | `boolean` | Enable/disable static file serving (default: true) |

The WebSocket endpoint is always available at `/ws`.

### Custom Web Page

By default, terminal-http serves a built-in HTML page with xterm.js. To use your own web page:

```java
// Use custom resources from your classpath
NettyWebsocketTtyBootstrap bootstrap = new NettyWebsocketTtyBootstrap()
        .setHost("localhost")
        .setPort(8080)
        .setResourcePath("/com/myapp/web");  // Your classpath location
```

Place your `index.html` at `src/main/resources/com/myapp/web/index.html`.

### Accessing Client Capabilities

After the client sends the `init` message, you can access the reported capabilities:

```java
import org.aesh.terminal.http.HttpTtyConnection;
import org.aesh.terminal.http.HttpDevice;

private void handleConnection(Connection connection) {
    if (connection instanceof HttpTtyConnection) {
        HttpTtyConnection httpConn = (HttpTtyConnection) connection;

        // Wait for init (or check isInitialized())
        HttpDevice device = (HttpDevice) httpConn.device();

        // Access client-reported capabilities
        String termType = device.type();              // "xterm-256color"
        String colorDepth = device.getReportedColorDepth();  // "TRUE_COLOR"
        List<String> features = device.getFeatures(); // ["UNICODE", "CLIPBOARD"]
        String userAgent = device.getUserAgent();

        // Check for specific features
        if (device.hasFeature("CLIPBOARD")) {
            // Client supports clipboard operations
        }
    }

    // Continue with readline...
}
```

### WebSocket-Only Mode

For applications that serve HTML from a separate web server:

```java
// Disable static file serving - only handle WebSocket at /ws
NettyWebsocketTtyBootstrap bootstrap = new NettyWebsocketTtyBootstrap()
        .setHost("localhost")
        .setPort(8080)
        .setServeStaticFiles(false);
```

### Browser Client with xterm.js

Create an HTML client using xterm.js with FitAddon for dynamic resizing:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Web Terminal</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@xterm/xterm@5.5.0/css/xterm.css" />
    <style>
        html, body {
            margin: 0;
            padding: 0;
            background: #1e1e1e;
            height: 100vh;
            overflow: hidden;
        }
        .wrapper {
            display: flex;
            flex-direction: column;
            height: 100vh;
            padding: 20px;
            box-sizing: border-box;
        }
        h1 {
            margin: 0 0 10px 0;
            font-size: 24px;
            color: #f0f0f0;
            text-align: center;
            flex-shrink: 0;
        }
        #terminal-container {
            flex: 1;
            border: 2px solid #444;
            border-radius: 8px;
            overflow: hidden;
            padding: 10px;
            background: #000;
            min-height: 0;
        }
        .connection-status {
            margin-bottom: 10px;
            padding: 8px 16px;
            border-radius: 4px;
            font-size: 14px;
            display: inline-block;
            align-self: center;
        }
        .status-connected { background: #1a472a; color: #4ade80; }
        .status-disconnected { background: #4a1a1a; color: #f87171; }
        .status-connecting { background: #4a3a1a; color: #fbbf24; }
    </style>
</head>
<body>
    <div class="wrapper">
        <h1>Web Terminal</h1>
        <div id="status" class="connection-status status-connecting">Connecting...</div>
        <div id="terminal-container"></div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/@xterm/xterm@5.5.0/lib/xterm.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@xterm/addon-fit@0.10.0/lib/addon-fit.js"></script>
    <script>
        (function() {
            'use strict';

            var statusEl = document.getElementById('status');

            function setStatus(status, message) {
                statusEl.className = 'connection-status status-' + status;
                statusEl.textContent = message;
            }

            // Detect terminal capabilities
            function detectCapabilities() {
                return {
                    type: 'xterm-256color',
                    colorDepth: 'TRUE_COLOR',
                    features: navigator.clipboard ? ['UNICODE', 'CLIPBOARD'] : ['UNICODE'],
                    userAgent: navigator.userAgent
                };
            }

            function connect() {
                var wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
                var wsHost = window.location.host || 'localhost:8080';
                var wsUrl = wsProtocol + '//' + wsHost + '/ws';

                setStatus('connecting', 'Connecting to ' + wsUrl + '...');

                var socket = new WebSocket(wsUrl);
                var term = null;
                var fitAddon = null;
                var resizeTimeout = null;

                socket.onopen = function() {
                    setStatus('connected', 'Connected');

                    term = new Terminal({
                        cursorBlink: true,
                        cursorStyle: 'block',
                        fontFamily: '"DejaVu Sans Mono", monospace',
                        fontSize: 14,
                        theme: {
                            background: '#000000',
                            foreground: '#f0f0f0',
                            cursor: '#f0f0f0'
                        }
                    });

                    // Load FitAddon for dynamic resizing
                    fitAddon = new FitAddon.FitAddon();
                    term.loadAddon(fitAddon);

                    term.open(document.getElementById('terminal-container'));
                    fitAddon.fit();

                    // Send init message with capabilities
                    var caps = detectCapabilities();
                    caps.cols = term.cols;
                    caps.rows = term.rows;
                    socket.send(JSON.stringify({
                        action: 'init',
                        type: caps.type,
                        colorDepth: caps.colorDepth,
                        features: caps.features,
                        cols: caps.cols,
                        rows: caps.rows,
                        userAgent: caps.userAgent
                    }));

                    socket.onmessage = function(event) {
                        term.write(event.data);
                    };

                    term.onData(function(data) {
                        socket.send(JSON.stringify({action: 'read', data: data}));
                    });

                    // Send resize events to server
                    term.onResize(function(size) {
                        socket.send(JSON.stringify({
                            action: 'resize',
                            cols: size.cols,
                            rows: size.rows
                        }));
                    });

                    // Handle window resize with debouncing
                    function handleResize() {
                        if (resizeTimeout) clearTimeout(resizeTimeout);
                        resizeTimeout = setTimeout(function() {
                            if (fitAddon) fitAddon.fit();
                        }, 100);
                    }
                    window.addEventListener('resize', handleResize);

                    socket.onclose = function(event) {
                        setStatus('disconnected', 'Disconnected (code: ' + event.code + ')');
                        term.write('\r\n\x1b[31mConnection closed.\x1b[0m\r\n');
                        window.removeEventListener('resize', handleResize);
                    };

                    socket.onerror = function(error) {
                        setStatus('disconnected', 'Connection error');
                    };

                    term.focus();
                };

                socket.onerror = function() {
                    setStatus('disconnected', 'Failed to connect');
                };
            }

            window.addEventListener('load', connect);
        })();
    </script>
</body>
</html>
```

This example includes:
- **FitAddon** for automatic terminal resizing
- **Capability detection** sent via `init` action
- **Resize event handling** with debouncing
- **Responsive layout** using flexbox

### Message Protocol

The WebSocket uses JSON messages for communication:

**Client to Server - Initialization (sent on connect):**
```json
{
  "action": "init",
  "type": "xterm-256color",
  "colorDepth": "TRUE_COLOR",
  "features": ["UNICODE", "CLIPBOARD"],
  "cols": 120,
  "rows": 40,
  "userAgent": "Mozilla/5.0 ..."
}
```

The `init` action reports client capabilities:
- `type` - Terminal type (e.g., "xterm-256color")
- `colorDepth` - Color support ("TRUE_COLOR", "256", "16")
- `features` - Supported features array
- `cols`/`rows` - Initial terminal dimensions
- `userAgent` - Browser identification

**Client to Server - User Input:**
```json
{"action": "read", "data": "user input here"}
```

**Client to Server - Terminal Resize:**
```json
{"action": "resize", "cols": 100, "rows": 30}
```

**Server to Client (output):**
Plain text with ANSI escape sequences for terminal rendering.

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
        NettyWebsocketTtyBootstrap http = new NettyWebsocketTtyBootstrap()
                .setPort(8080);

        // Start all servers
        ssh.start();
        telnet.start();
        http.start(MultiProtocolServer::handleConnection);

        System.out.println("Servers started:");
        System.out.println("  SSH:       ssh -p 2222 user@localhost");
        System.out.println("  Telnet:    telnet localhost 2323");
        System.out.println("  WebSocket: http://localhost:8080");
        
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
// Start servers
ssh.start();
telnet.start();
http.start(connectionHandler);  // WebSocket requires handler

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
    try {
        http.stop().get(5, TimeUnit.SECONDS);
    } catch (Exception e) {
        // Handle timeout
    }

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
