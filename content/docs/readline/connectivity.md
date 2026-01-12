---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Remote Connectivity'
weight: 10
---

Æsh Readline supports remote connections via SSH, Telnet, and WebSockets.

## SSH Connectivity

### Adding SSH Dependency

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-ssh</artifactId>
  <version>3.5</version>
</dependency>
```

### SSH Server Example

```java
import org.aesh.terminal.ssh.SshTerminal;

SshTerminal ssh = SshTerminal.builder()
        .host("0.0.0.0")
        .port(2222)
        .keyPair(new File("hostkey.ser"))
        .connectionHandler(connection -> {
            Readline readline = ReadlineBuilder.builder().build();
            read(connection, readline, "[ssh]$ ");
        })
        .build();

ssh.start();
```

## Telnet Connectivity

### Adding Telnet Dependency

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-telnet</artifactId>
  <version>3.5</version>
</dependency>
```

### Telnet Server Example

```java
import org.aesh.terminal.telnet.TelnetTerminal;

TelnetTerminal telnet = TelnetTerminal.builder()
        .host("0.0.0.0")
        .port(2323)
        .connectionHandler(connection -> {
            Readline readline = ReadlineBuilder.builder().build();
            read(connection, readline, "[telnet]$ ");
        })
        .build();

telnet.start();
```

## HTTP/WebSocket Connectivity

### Adding HTTP Dependency

```xml
<dependency>
  <groupId>org.aesh</groupId>
  <artifactId>terminal-http</artifactId>
  <version>3.5</version>
</dependency>
```

### WebSocket Terminal Example

```java
import org.aesh.terminal.http.WebSocketTerminal;

WebSocketTerminal ws = WebSocketTerminal.builder()
        .host("0.0.0.0")
        .port(8080)
        .path("/terminal")
        .connectionHandler(connection -> {
            Readline readline = ReadlineBuilder.builder().build();
            read(connection, readline, "[web]$ ");
        })
        .build();

ws.start();
```

## Common Connection Pattern

All remote terminals follow a similar pattern:

```java
import org.aesh.readline.Readline;
import org.aesh.readline.ReadlineBuilder;

private static void read(Connection connection, Readline readline, String prompt) {
    readline.readline(connection, prompt, input -> {
        if (input != null && input.equals("exit")) {
            connection.write("Goodbye!\n");
            connection.close();
        }
        else {
            connection.write("Echo: " + input + "\n");
            read(connection, readline, prompt);
        }
    });
}
```

## Host Keys (SSH)

SSH requires host key for secure connections:

```java
import org.apache.sshd.common.keyprovider.KeyPairProvider;
import org.apache.sshd.server.keyprovider.SimpleGeneratorHostKeyProvider;

KeyPairProvider keyPairProvider = new SimpleGeneratorHostKeyProvider(
    new File("hostkey.ser").toPath()
);

SshTerminal ssh = SshTerminal.builder()
        .keyPair(keyPairProvider)
        // ...
        .build();
```

## Authentication (SSH)

Configure SSH authentication:

```java
import org.apache.sshd.server.auth.password.PasswordAuthenticator;
import org.apache.sshd.server.auth.pubkey.PublickeyAuthenticator;

PasswordAuthenticator passwordAuth = (username, password, session) -> {
    return authenticate(username, password);
};

PublickeyAuthenticator publicKeyAuth = (username, key, session) -> {
    return authenticatePublicKey(username, key);
};

SshTerminal ssh = SshTerminal.builder()
        .passwordAuthenticator(passwordAuth)
        .publickeyAuthenticator(publicKeyAuth)
        // ...
        .build();
```

## Starting/Stopping Servers

```java
// Start server
ssh.start();
telnet.start();
ws.start();

// Stop server
ssh.stop();
telnet.stop();
ws.stop();

// Check if running
boolean isRunning = ssh.isRunning();
```

## Multiple Connection Types

Run multiple connection types simultaneously:

```java
SshTerminal ssh = SshTerminal.builder()...build();
TelnetTerminal telnet = TelnetTerminal.builder()...build();

ssh.start();
telnet.start();

// Both run concurrently
```

## Working Examples

The [aesh-examples repository](https://github.com/aeshell/aesh-examples) contains complete working examples for all connectivity types:

- **[shell-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-ssh)** - Shell over SSH with authentication
- **[shell-telnet](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-telnet)** - Shell over Telnet
- **[shell-websocket](https://github.com/aeshell/aesh-examples/tree/master/readline/shell-websocket)** - Web-based terminal over WebSocket
- **[cmd-mirror-ssh](https://github.com/aeshell/aesh-examples/tree/master/readline/cmd-mirror-ssh)** - Command mirroring over SSH

See the [Examples and Tutorials](../examples) page for detailed information about all available examples, including setup instructions and complete source code.
