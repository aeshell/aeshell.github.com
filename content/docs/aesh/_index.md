---
date: '2026-01-11T15:00:00+01:00'
draft: false
title: 'Æsh'
---

Æsh is a Java library to easily create commands through a well-defined API.

## Overview

Æsh takes care of all parsing and injection for your commands. It uses annotations as the default way of adding metadata for commands, but also provides a builder API if that is preferred.

## Key Features

- Easy to use API to create everything from simple to advanced commands
- Supports different types of options (list, group, single) and arguments
- Built-in completors for default values, booleans and files
- Multiple hierarchy of sub commands (e.g., `git rebase/pull`)
- Automatic injection of option values and arguments during execution
- Custom validators, activators, completors, converters, renderers and parsers
- Automatically generates help/info text based on provided metadata
- Add and remove commands during runtime
