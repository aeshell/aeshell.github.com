---
date: '2026-01-12T14:00:00+01:00'
draft: false
title: 'Built-in Commands'
weight: 15
---

The `aesh-builtins` module provides ready-to-use POSIX-like command implementations for common shell operations. Instead of building basic commands from scratch, you can register these pre-built, tested commands in your CLI application.

## Installation

### Maven

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-builtins</artifactId>
    <version>3.17.5</version>
</dependency>
```

### Gradle

```groovy
implementation 'org.aesh:aesh-builtins:3.17.5'
```

## Available Commands

### File System Commands

| Command | Class | Description |
|---------|-------|-------------|
| `cat` | `org.aesh.builtins.cat.Cat` | Display file contents |
| `cd` | `org.aesh.builtins.cd.Cd` | Change working directory |
| `ls` | `org.aesh.builtins.ls.Ls` | List directory contents with colors |
| `mkdir` | `org.aesh.builtins.mkdir.Mkdir` | Create directories |
| `mv` | `org.aesh.builtins.mv.Mv` | Move or rename files |
| `rm` | `org.aesh.builtins.rm.Rm` | Remove files and directories |
| `touch` | `org.aesh.builtins.touch.Touch` | Create files or update timestamps |
| `pwd` | `org.aesh.builtins.pwd.Pwd` | Print working directory |
| `pushd` | `org.aesh.builtins.pushd.Pushd` | Push directory onto stack |
| `popd` | `org.aesh.builtins.pushd.Popd` | Pop directory from stack |

### Text Processing

| Command | Class | Description |
|---------|-------|-------------|
| `grep` | `org.aesh.builtins.grep.Grep` | Search text with regex, colored output |
| `echo` | `org.aesh.builtins.echo.Echo` | Print text to output |

### Display

| Command | Class | Description |
|---------|-------|-------------|
| `less` | `org.aesh.builtins.less.Less` | File viewer with paging, search |
| `clear` | `org.aesh.builtins.clear.Clear` | Clear terminal screen |

## Usage

Register the commands you need in your application:

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.builtins.cat.Cat;
import org.aesh.builtins.cd.Cd;
import org.aesh.builtins.clear.Clear;
import org.aesh.builtins.echo.Echo;
import org.aesh.builtins.grep.Grep;
import org.aesh.builtins.less.Less;
import org.aesh.builtins.ls.Ls;
import org.aesh.builtins.mkdir.Mkdir;
import org.aesh.builtins.pwd.Pwd;
import org.aesh.builtins.rm.Rm;

public class MyShell {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                .command(Cat.class)
                .command(Cd.class)
                .command(Clear.class)
                .command(Echo.class)
                .command(Grep.class)
                .command(Less.class)
                .command(Ls.class)
                .command(Mkdir.class)
                .command(Pwd.class)
                .command(Rm.class)
                // Add your own commands here
                .command(MyCommand.class)
                .addExitCommand()
                .prompt("[myshell]$ ")
                .start();
    }
}
```

You can register all of them or just the ones your application needs.

## Command Details

### cat

Display one or more file contents:

```
[myshell]$ cat file1.txt file2.txt
```

### grep

Search for patterns in piped input or (when piped) standard input:

```
[myshell]$ grep -i "error" /var/log/app.log
[myshell]$ cat file.txt | grep pattern
```

Options: `-i` (case insensitive), `-c` (count matches), `-l` (list filenames), `-n` (line numbers), `-e` (pattern).

### less

View files with paging, search, and navigation:

```
[myshell]$ less large-file.txt
```

Key bindings:
- **Space / PgDn**: Page down
- **b / PgUp**: Page up
- **Enter / Down**: Line down
- **Up**: Line up
- **/pattern**: Search forward
- **n / N**: Next / previous match
- **g / G**: Go to start / end
- **h**: Help
- **q**: Quit

Less also reads from piped input:

```
[myshell]$ cat file.txt | less
```

### ls

List directory contents with color-coded file types:

```
[myshell]$ ls
[myshell]$ ls -la /tmp
```

### pushd / popd

Directory stack for quick navigation:

```
[myshell]$ pushd /tmp
[myshell]$ pushd /var/log
[myshell]$ popd           # returns to /tmp
[myshell]$ popd           # returns to original directory
```

## Legacy Extensions

The [aesh-extensions](https://github.com/aeshell/aesh-extensions) repository contains additional commands that have not been moved to `aesh-builtins`, including `matrix` (terminal animation) and `harlem` (audio demo). These require the legacy `aesh-extensions` artifact:

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-extensions</artifactId>
    <version>1.8</version>
</dependency>
```
