---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Æsh, Another Extendable SHell'
---

Æsh is a collection of Java libraries for building powerful command-line interfaces and console applications.

## Projects

### Æsh
A library to easily create commands through a well-defined API. Æsh handles all parsing, validation, and injection for your commands.

### Æsh Readline
A readline library supporting most GNU Readline features, including line editing, history, completion, masking, and remote connectivity.

## Features

### Æsh
- Easy to use API to create everything from simple to advanced commands
- Support for different types of options (list, group, single) and arguments
- Built-in completors for default values, booleans and files
- Multiple hierarchy of sub commands (eg: `git rebase/pull`)
- Automatic injection of option values and arguments during execution
- Custom validators, activators, completors, converters, renderers and parsers
- Automatically generates help/info text based on metadata
- Add and remove commands during runtime

### Æsh Readline
- Line editing with undo/redo
- History (search, persistence)
- Completion support
- Password masking
- Paste buffer
- Emacs and Vi editing modes
- Cross-platform (POSIX and Windows)
- Configurable history file & buffer size
- Standard out and standard error support
- Redirect, alias, and pipeline support
- Remote connectivity (SSH, Telnet, WebSockets)
