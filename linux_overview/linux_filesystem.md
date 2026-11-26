---
layout: series
title: The Linux Filesystem
permalink: /linux-overview/linux-filesystem/
series: Linux for Security Analysts
weight: 2.0
---

## Overview

* Is "Everything" really a file?
* Types of Files

## "Everything" is a File

The phrase "everything is a file" is technically true and false, depending on your definition of what a "file" is.
On the Linux kernel, a file is more of a shared interface than a distinct data blob, such as a text or binary file.

This aligns quite well with the UNIX guideline of "write programs to work together".
A shared API, such as a filesystem API, allows the same tool to access various types of data. An easy example of this would be the `cat` command.

If everything is a file, `cat` can display text files (traditional):

```sh
$ cat README.md
# ch0mler.github.io

Welcome to my blog...<snip>
```

`cat` can display process information from the kernel:

```sh
$ cat /proc/self/cmdline
cat/proc/self/cmdline
```

It can display data from other processes using a named pipe:

```sh
$ sh -c 'mknod data p; sleep 10; echo "test" > data' &
[1] 260237
$ cat data
test
```

All of this can happen because `cat` is built to read files, and files are an extremely flexible object in the Linux kernel.
The operating system even provides detailed system information by exposing virtual filesystems like `/proc` (processes) and `/dev` (devices).

### File Metadata and inodes

## Types of Files

### Regular Files

### Directories

### Symbolic Links and Hard Links

### Devices

### Pipes and Named Pipes

### Sockets

## Further Reading

My friend and former coworker Dave Bogle wrote an excellent deep dive into the Linux virtual filesystem, and what it means for defenders here:

[https://www.zscaler.com/blogs/cybersecurity-best-practices/linux-vfs](https://www.zscaler.com/blogs/cybersecurity-best-practices/linux-vfs)
