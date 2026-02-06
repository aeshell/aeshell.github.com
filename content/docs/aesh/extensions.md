---
date: '2026-01-12T14:00:00+01:00'
draft: false
title: 'Extensions Library'
weight: 15
---

The [aesh-extensions](https://github.com/aeshell/aesh-extensions) repository provides ready-to-use command implementations for common shell operations. Instead of building basic commands from scratch, you can use these pre-built, tested commands in your CLI applications.

## Why Use Extensions?

**Save Development Time**
- Pre-built implementations of common shell commands
- Battle-tested code used in production systems
- Focus on your application logic, not basic file operations

**Learn Best Practices**
- See how experienced developers implement Æsh commands
- Study real-world examples of options, arguments, and validation
- Use as templates for your own custom commands

**Build Functional Shells Quickly**
- Get a working shell with `ls`, `cd`, `cat`, etc. in minutes
- Combine with your custom commands
- Professional command implementations out of the box

## Installation

### Maven

```xml
<dependency>
    <groupId>org.aesh</groupId>
    <artifactId>aesh-extensions</artifactId>
    <version>1.7</version>
</dependency>
```

### Gradle

```groovy
implementation 'org.aesh:aesh-extensions:1.7'
```

## Available Commands

### File System Commands

**`cat`** - Concatenate and display file contents
- Display one or multiple files
- Options for line numbering
- Standard Unix `cat` behavior

**`cd`** - Change directory
- Navigate the file system
- Support for relative and absolute paths
- Home directory (`~`) support

**`ls`** - List directory contents
- List files and directories
- Multiple display formats
- File attribute information

**`pwd`** - Print working directory
- Show current directory path
- Standard Unix behavior

**`mkdir`** - Create directories
- Create single or multiple directories
- Parent directory creation option
- Permission handling

**`rm`** - Remove files and directories
- Delete files and directories
- Recursive deletion option
- Safety checks and confirmations

**`mv`** - Move/rename files
- Move files between directories
- Rename files and folders
- Overwrite protection

**`less`** - View file contents with pagination
- Scroll through large files
- Search within files
- Navigation controls

**`more`** - Simple file pager
- Basic file viewing
- Page-by-page navigation
- Classic Unix `more` behavior

### Text Processing Commands

**`grep`** - Search text patterns
- Regular expression search
- File content searching
- Pattern matching options

**`echo`** - Display text
- Print messages to output
- Variable expansion
- Standard echo functionality

### Terminal Commands

**`clear`** - Clear the terminal screen
- Clear display
- Reset cursor position
- Standard terminal behavior

**`exit`** - Exit the shell
- Clean shell termination
- Exit code support
- Cleanup on exit

### Navigation Commands

**`pushdpopd`** - Directory stack navigation
- Push directories onto stack
- Pop directories from stack
- Navigate directory history

### Interactive Commands

**`choice`** - Prompt for user selection
- Present multiple options
- Get user input
- Validation support

### Fun/Demo Commands

**`matrix`** - Matrix-style screen effect
- Terminal animation
- Demonstration of terminal control
- Visual effects example

**`harlem`** - Harlem Shake animation
- Terminal animation demo
- Shows advanced terminal capabilities
- Entertainment command

**`highlight`** - Syntax highlighting
- Display code with syntax coloring
- Multiple language support
- Customizable color schemes

**`groovy`** - Groovy script execution
- Execute Groovy scripts
- Interactive Groovy REPL
- Scripting integration

**`example`** - Example command template
- Template for creating new commands
- Shows command structure
- Learning resource

## Using Extensions in Your Application

### Quick Start - Add All Commands

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.extensions.cat.Cat;
import org.aesh.extensions.cd.Cd;
import org.aesh.extensions.ls.Ls;
import org.aesh.extensions.pwd.Pwd;
import org.aesh.extensions.clear.Clear;
import org.aesh.extensions.exit.Exit;

public class MyShell {
    public static void main(String[] args) {
        AeshConsoleRunner.builder()
                // Add extension commands
                .command(Ls.class)
                .command(Cd.class)
                .command(Pwd.class)
                .command(Cat.class)
                .command(Clear.class)
                .command(Exit.class)
                // Add your custom commands
                .command(MyCustomCommand.class)
                .prompt("[myshell]$ ")
                .start();
    }
}
```

### Selective Command Usage

You don't have to use all commands. Pick only what you need:

```java
// Minimal shell with just navigation
AeshConsoleRunner.builder()
        .command(Cd.class)
        .command(Pwd.class)
        .command(Ls.class)
        .command(Exit.class)
        .prompt("[minimal]$ ")
        .start();
```

### Combining with Custom Commands

Mix extension commands with your own:

```java
// File management shell
AeshConsoleRunner.builder()
        // Extension commands for basic operations
        .command(Ls.class)
        .command(Cat.class)
        .command(Rm.class)
        .command(Mv.class)
        // Your custom file processing commands
        .command(EncryptFileCommand.class)
        .command(CompressFileCommand.class)
        .command(AnalyzeFileCommand.class)
        .prompt("[filetools]$ ")
        .start();
```

## Command Examples

### Using `ls` Command

The `ls` extension provides familiar Unix-style directory listings:

```bash
[myshell]$ ls
file1.txt  file2.txt  docs/  src/

[myshell]$ ls -l
-rw-r--r--  1024  file1.txt
-rw-r--r--  2048  file2.txt
drwxr-xr-x  4096  docs/
drwxr-xr-x  8192  src/
```

### Using `cat` Command

Display file contents:

```bash
[myshell]$ cat readme.txt
This is the contents of the file.
Multiple lines are displayed.

[myshell]$ cat file1.txt file2.txt
Contents of file1
Contents of file2
```

### Using `cd` and `pwd` Commands

Navigate directories:

```bash
[myshell]$ pwd
/home/user/projects

[myshell]$ cd src
[myshell]$ pwd
/home/user/projects/src

[myshell]$ cd ..
[myshell]$ pwd
/home/user/projects
```

## Customizing Extension Commands

While you can use extension commands as-is, you can also extend or customize them:

### Extending an Extension Command

```java
import org.aesh.extensions.ls.Ls;
import org.aesh.command.CommandResult;
import org.aesh.command.invocation.CommandInvocation;

public class CustomLs extends Ls {
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Add custom behavior before
        invocation.println("Listing files...");
        
        // Call parent implementation
        CommandResult result = super.execute(invocation);
        
        // Add custom behavior after
        invocation.println("Done listing files.");
        
        return result;
    }
}
```

### Creating Commands Based on Extensions

Use extension commands as templates:

```java
// Study the Cat command source as a template
// Then create your own file display command with custom formatting
@CommandDefinition(name = "show", description = "Custom file display")
public class ShowCommand implements Command<CommandInvocation> {
    
    @Argument(description = "File to show", required = true)
    private File file;
    
    @Override
    public CommandResult execute(CommandInvocation invocation) {
        // Your custom implementation
        // Based on patterns learned from Cat command
        return CommandResult.SUCCESS;
    }
}
```

## Source Code as Learning Material

The extension commands are excellent learning resources:

**Study Command Structure**
- See how options and arguments are defined
- Learn error handling patterns
- Understand file operation best practices

**Analyze Implementation Patterns**
- Proper validation techniques
- User feedback and error messages
- Resource cleanup and management

**Use as Templates**
- Copy structure for your commands
- Adapt patterns to your needs
- Follow established conventions

## Example: Building a File Manager

Here's a complete example using extensions to build a file manager shell:

```java
import org.aesh.AeshConsoleRunner;
import org.aesh.extensions.cat.Cat;
import org.aesh.extensions.cd.Cd;
import org.aesh.extensions.clear.Clear;
import org.aesh.extensions.exit.Exit;
import org.aesh.extensions.ls.Ls;
import org.aesh.extensions.mkdir.Mkdir;
import org.aesh.extensions.mv.Mv;
import org.aesh.extensions.pwd.Pwd;
import org.aesh.extensions.rm.Rm;
import org.aesh.extensions.grep.Grep;
import org.aesh.extensions.less.Less;

public class FileManager {
    
    public static void main(String[] args) {
        System.out.println("File Manager Shell");
        System.out.println("==================");
        System.out.println("Available commands:");
        System.out.println("  ls, cd, pwd, cat, less, grep");
        System.out.println("  mkdir, rm, mv");
        System.out.println("  clear, exit");
        System.out.println();
        
        AeshConsoleRunner.builder()
                // Navigation
                .command(Ls.class)
                .command(Cd.class)
                .command(Pwd.class)
                // Viewing
                .command(Cat.class)
                .command(Less.class)
                .command(Grep.class)
                // File Operations
                .command(Mkdir.class)
                .command(Rm.class)
                .command(Mv.class)
                // Utility
                .command(Clear.class)
                .command(Exit.class)
                .prompt("[filemgr]$ ")
                .start();
    }
}
```

Running this gives you a fully functional file manager:

```bash
File Manager Shell
==================
Available commands:
  ls, cd, pwd, cat, less, grep
  mkdir, rm, mv
  clear, exit

[filemgr]$ pwd
/home/user

[filemgr]$ mkdir testdir
Directory created: testdir

[filemgr]$ cd testdir
[filemgr]$ pwd
/home/user/testdir

[filemgr]$ ls
[filemgr]$ 

[filemgr]$ exit
Goodbye!
```

## Architecture Notes

### Command Independence

Each extension command is independent and can be used separately:
- No dependencies between commands
- Use only what you need
- Small footprint

### File System Operations

File system commands respect:
- Current working directory context
- File permissions
- Platform-specific path separators
- Symbolic links (where applicable)

### Terminal Integration

Commands that need terminal control (like `clear`, `less`, `matrix`) work with:
- Local terminals
- SSH connections
- Telnet connections
- WebSocket terminals

## Testing with Extensions

Extension commands are great for testing your shell infrastructure:

```java
@Test
public void testShellWithExtensions() {
    AeshRuntimeRunner runner = AeshRuntimeRunner.builder()
            .command(Ls.class)
            .command(Pwd.class)
            .build();
    
    // Test that ls works
    String result = runner.execute("ls");
    assertNotNull(result);
    
    // Test pwd
    result = runner.execute("pwd");
    assertTrue(result.contains("/"));
}
```

## Best Practices

### Command Selection

**Do:**
- Add only commands your users need
- Consider your application's purpose
- Keep the command set focused

**Don't:**
- Add every command "just in case"
- Expose dangerous operations without safeguards
- Overwhelm users with too many commands

### Security Considerations

**Be Cautious With:**
- `rm` - Can delete files permanently
- `groovy` - Executes arbitrary code
- File write operations - Can modify system

**Recommendations:**
- Add confirmation prompts for destructive operations
- Limit file system access to specific directories
- Consider user permissions in multi-user environments
- Audit command usage in security-sensitive applications

### Customization Strategy

**When to Extend:**
- Need to add logging
- Want to restrict functionality
- Require additional validation
- Need integration with your app

**When to Use As-Is:**
- Standard behavior is sufficient
- Rapid prototyping
- Testing and development
- Simple use cases

## Contributing to Extensions

Found a bug or want to add a command? The extensions repository welcomes contributions:

- **Repository:** [github.com/aeshell/aesh-extensions](https://github.com/aeshell/aesh-extensions)
- **Issues:** [Report bugs or request commands](https://github.com/aeshell/aesh-extensions/issues)
- **Pull Requests:** Submit improvements or new commands

## Additional Resources

- **[Examples Repository](https://github.com/aeshell/aesh-examples)** - See extensions in use
- **[Command Definition Guide](../command-definition)** - Learn to create your own commands
- **[Options and Arguments](../options)** - Understanding command parameters
- **[Examples and Tutorials](../examples)** - Complete working examples

## Next Steps

1. **Add the dependency** to your project
2. **Start with basic commands** (ls, cd, pwd, exit)
3. **Test in your shell** to ensure they work
4. **Add more commands** as needed
5. **Customize** extension commands for your specific needs
6. **Study the source code** to learn command implementation patterns

The extensions library turns Æsh from a command framework into a complete shell toolkit, dramatically reducing the time needed to build functional CLI applications!
