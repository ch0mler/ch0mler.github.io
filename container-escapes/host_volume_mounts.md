---
layout: series
title: Host Volume Mount Escapes
permalink: /container-escapes/host-volume-mounts
series: Container Escapes
weight: 4.0
---

## Escaping the Container

To test this out, I spawned a host with a volume mounted to my home directory.
Testing is straightforward - create some files inside the volume mount and examine the `open*` family of syscalls.

```sh
# start a container with the root filesystem mounted to /host
$ docker run --rm -it -v /home/ubuntu:/root -v /:/host ubuntu
# use the 'echo' shell built-in to modify a monitored file on the host
root@e096671bca2c:/$ echo test > /host/tmp/breakout_test.txt
# use the 'echo' binary to modify a monitored file on the host
root@e096671bca2c:/$ /usr/bin/echo test > /host/tmp/breakout_test.txt
```

We can also write a really simple C program to perform the same activity, since we don't need built-in tools to write files.

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    char cwd[1024];
    // Get and print current working directory
    if (getcwd(cwd, sizeof(cwd)) != NULL) {
        printf("%s\n", cwd);
    } else {
        perror("getcwd() error");
        return 1;
    }

    // Open file for writing
    FILE *ofile = fopen("/host/tmp/breakout_test.txt", "w");
    if (ofile == NULL) {
        perror("Error opening file");
        return 1;
    }

    // Write to file
    fprintf(ofile, "breakouttest");

    // Close file
    fclose(ofile);

    return 0;
}
```

After compiling it in the container:

```sh
root@e096671bca2c:/$ gcc -o breakout_test ./breakout.c && chmod +x breakout_test
root@e096671bca2c:/$ ./breakout_test
```

## Detection

The `auditd` framework can monitor the usual file open/create/modify syscalls, as well as watching changes to specific paths via inode monitoring.
This is a difficult thing to scale, which is why nearly every Linux EDR filters the areas of the filesystem that it monitors.
Additionally, the maxim that [everything is a file](/linux-overview/linux-filesystem) means a lot of these syscalls are used for more than reading and writing regular files.

### How It Works

If a container has access to the host via a volume mount, it shouldn't be too surprising that said container can read/write to it.
The implications of sharing filesystems can be more dangerous than it initially appears though, especially since so many subsystems allow interaction via virtual filesystems (e.g. reading environment variables from the `/proc` filesystem).

The procedures may change slightly from using a shell built-in (`echo`) to a standard binary (`/usr/bin/echo`)
to a custom binary (`/breakout_test`), but the core audit logs overlap in the most important part - the `openat` syscall and `PATH` log types.

One interesting thing to note about these audit logs is that the filesystem *from the container's point of view* is what is logged.
In each of the records below, we can see that PATH entries have `name=/host/tmp/breakout_test.txt` rather than `/tmp/breakout_test.txt`.

#### Echo Built-ins

Relevant audit logs:

```sh
type=PROCTITLE msg=audit(06/22/26 17:18:24) : proctitle=/bin/bash
type=PATH msg=audit(06/22/26 17:18:24) : item=0 name=/host/tmp/breakout_test.txt inode=1079
    dev=ca:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=NORMAL
    cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/22/26 17:18:24) : cwd=/
type=SYSCALL msg=audit(06/22/26 17:18:24) : arch=x86_64 syscall=openat success=yes exit=3
    a0=AT_FDCWD a1=0x61a6e1ce8960 a2=O_WRONLY|O_CREAT|O_TRUNC a3=0x1b6 items=1 ppid=2551 pid=2575
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=bash exe=/usr/bin/bash subj=docker-default key=test_filemod
```

Note the attribution of the `openat` syscall to `/bin/bash` instead of `/usr/bin/echo`. 
This is due to the file redirect `">"` being set up by the shell, and then handed off 
to the `echo` binary via process stream redirection.
See [this blog](https://zscaler.com/blogs/cybersecurity-best-practices/process-streams) 
for more details on shell redirects.

#### Echo Binary

The primary difference between Auditd logs for this and the shell built-in version is
the addition of an EXECVE and syscall with information about the command line.

```sh
type=PROCTITLE msg=audit(06/22/26 17:18:34) : proctitle=/bin/bash
type=PATH msg=audit(06/22/26 17:18:34) : item=0 name=/host/tmp/breakout_test.txt inode=1079
    dev=ca:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=NORMAL
    cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/22/26 17:18:34) : cwd=/
type=SYSCALL msg=audit(06/22/26 17:18:34) : arch=x86_64 syscall=openat success=yes exit=3
    a0=AT_FDCWD a1=0x61a6e1ce8270 a2=O_WRONLY|O_CREAT|O_TRUNC a3=0x1b6 items=1 ppid=2575 pid=2620
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=bash exe=/usr/bin/bash subj=docker-default key=test_filemod
```

```sh
type=PROCTITLE msg=audit(06/22/26 17:18:34) : proctitle=/bin/bash
type=PATH msg=audit(06/22/26 17:18:34) : item=1 name=/lib64/ld-linux-x86-64.so.2 inode=519849
    dev=00:27 modoe=file,755 ouid=root ogid=root rdev=00:00 nametype=NORMAL
    cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(06/22/26 17:18:34) : item=0 name=/usr/bin/echo inode=519127
    dev=00:27 modoe=file,755 ouid=root ogid=root rdev=00:00 nametype=NORMAL
    cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/22/26 17:18:34) : cwd=/
type=EXECVE msg=audit(06/22/26 17:18:34) : argc=2 a0=/usr/bin/echo a1=test
type=SYSCALL msg=audit(06/22/26 17:18:34) : arch=x86_64 syscall=execve success=yes exit=0
    a0=0x61a6e1ce8430 a1=0x61a6e1ce8730 a2=0x61a6e1ce9bb0 a3=0x20 items=2 ppid=2575 pid=2620
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=echo exe=/usr/lib/cargo/bin/coreutils/echo subj=docker-default key=process_creation
```

Note the extra `EXECVE` log entry for `/usr/bin/echo` indicating an actual process was spawned as opposed to a shell built-in.

#### Custom Binary

This is a tricky one to detect specifically - largely because "random binary interacting with random file"
is a rat's nest of exclusions on the best of days. Note, however, that the `PATH` event below is reported
using the filepath *from the container's perspective*. Creating some simple rules for common
container escape directories, like `/host/`, and looking for those as path prefixes, gives us some visibility.
Directory watching on Auditd is a thing, but if you are monitoring containers which have known host mount paths,
this is still not likely your best bet due to expected noise across the Linux filesystem.

Relevant audit logs:

```sh
type=PROCTITLE msg=audit(06/22/26 17:29:08) : proctitle=./breakout_test
type=PATH msg=audit(06/22/26 17:29:08) : item=0 name=/host/tmp/breakout_test.txt inode=1079
    dev=ca:01 mode=file,644 ouid=root ogid=root rdev=00:00 nametype=NORMAL
    cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/22/26 17:29:08) : cwd=/
type=SYSCALL msg=audit(06/22/26 17:29:08) : arch=x86_64 syscall=openat success=yes exit=3
    a0=AT_FDCWD a1=0x620b08969015 a2=O_WRONLY|O_CREAT|O_TRUNC a3=0x1b6 items=1 ppid=3881 pid=3925
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=breakout_test exe=/breakout_test subj=docker-default key=test_filemod
```
