---
layout: series
title: Kernel Namespace Escapes
permalink: /container-escapes/kernel-namespaces
series: Container Escapes
weight: 1.0
---

## Escaping the Container

To test this out, I spawned a privileged container that shares a couple namespaces with the host.
Then I ran a few commands to prove the namespacres are shared, escaped to the host using `nsenter` to 
spawn a shell with the host's namespaces, and created a `netcat` listener to have a long-running process to examine.

```sh
# start up a new privileged container to allow the namespace escape
$ docker run --privileged --pid=host --userns=host --rm --it ubuntu
# the container is able to view the host processes
root@feea595a3c86:/$ ps aux
USER    PID %CPU    %MEM    VSZ     RSS     TTY STAT    START   TIME    COMMAND
root    1   0.0     0.1     22696   13972   ?   Ss      Jun22   0:08    /sbin/init
root    2   0.0     0.0     0       0       ?   S       Jun22   0:08    [kthreadd]
root    3   0.0     0.0     0       0       ?   S       Jun22   0:08    [pool_workqueue_release]
...<snip>...
# the container sees its own "docker-$CONTAINER_ID" cgroup as root
root@feea595a3c86:/$ cat /proc/self/cgroup
0::/
# create a new bash process to use the host's namespaces and escape the container
root@feea595a3c86:/$ nsenter --target 1 --mount --uts --ipc --net --pid -- bash -l
# in the escaped shell, look at the init process's cgroup - note the use of "../../"
# to indicate we're a higher level than the docker cgroup
root@HOSTNAME:/$ cat /proc/1/cgroup
0::/../../init.scope
# spawn a netcat listener as a long-running process
root@HOSTNAME:/$ nc -nvlp 8080
Listening on 0.0.0.0 8080
```

**Note**: While privileged containers are generally not recommended, there can be good reasons to use them.
Elevated access can be needed for certain resources, though [capabilities](https://www.man7.org/linux/man-pages/man7/capabilities.7.html)
are considered a better solution, and host monitoring tools deployed alongside the containers (EDR sensors)
generally need elevated permissions to monitor all containers at once.

From another terminal, we can review host the namespaces for each of these processes looks.

```sh
# view system cgroups to group all the container's processes
$ sudo systemd-cgls -u docker-feea595a3c86.scope
Unit docker-feea595a3c86.scope (/system.slice/docker-feea595a3c86.scope):
|--45467 /bin/bash
|--45566 nsenter -t 1 --mount --uts --ipc --net --pid -- bash -l
|--45567 bash -l
|--45663 nc -nvlp 8080
# verify the base shell spawned by our "docker run" command is only associated with the docker container
$ sudo cat /proc/45567/cgroup
0::/system.slice/docker-feea595a3c86.scope
# do the same for the netcat process spawned from our escaped shell
$ sudo cat /proc/45664/cgroup
0::/system.slice/docker-feea595a3c86.scope
# show the differences between namespaces in container and on host
# note how the PID and User namespaces are the same across processes, but
# the IPC,MNT,NET,UTS namespaces are all different in PID 45467, which is the only process restricted by the container
$ sudo ps -p 45516,45467,45566,45663 -o pid,args,ppid,ipcns,mntns,netns,pidns,userns,utsns
PID     COMMAND                 PPID    IPCNS       MNTNS       NETNS       PIDNS       USERNS      UTSNS
45467   /bin/bash               45440   4026532280  4026532278  4026532282  4026531836  4026531837  4026532279  # -- initial container shell spawned by docker
45516   -bash                   1301    4026531839  4026531832  4026531833  4026531836  4026531837  4026531838  # -- my host shell to examine container
45467   nsenter -t 1 --mount    45567   4026531839  4026531832  4026531833  4026531836  4026531837  4026531838  # -- nsenter command that performed the escape
45467   nc -nvlp 8080           45567   4026531839  4026531832  4026531833  4026531836  4026531837  4026531838  # -- netcat command spawned on host after escape
```

**Note**: The docker container ID was shortened in the above terminal for readability.

## Detection

The `auditd` framework can be set up to monitor namespace modifications by watching the `setns` and `unshare` syscalls.
These two syscalls are actually extremely common, especially on any system using containers (think K8s nodes, production build pipelines, or developer workstations), but namespace modification does leave some tell-tale signs in the audit trail.

### How It Works

We can create a new process and set its namespace using the `nsenter` tool fairly easily.
This escape works by spawning a new shell (could be any binary, but a shell is easy)
and using the `init` process's namespaces as a template for the shell's namespaces - complete with host access.

```sh
# --target 1 : get namespaces from the init process (root-owned)
# --mount --uts --ipc --net --pid : create separate namespaces for these resources
# bash -l : spawn a new shell with these elevated namespaces
$ nsenter --target 1 --mount --uts --ipc --net --pid -- bash -l
```

We'd see PROCTITLE and SYSCALL entries such as the following:

```sh
type=PROCTITLE msg=audit(06/22/26 16:58:16) : proctitle=nsenter --target 1 --mount --uts --ipc --net --pid -- bash -l
type=SYSCALL msg=audit(06/22/26 16:58:16) : arch=x86_64 syscall=setns success=yes exit=0 a0=0x3
    a1=CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWPID|CLONE_NEWNET a2=0x0 a3=0x6c020000 items=1 ppid=46452 pid=46497
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=nsenter exec=/usr/bin/nsenter subj=unconfined key=container-config
```

### Typical setns syscall

In contrast to this escape attempt, most `setns` calls come from container daemons
creating new containers - `dockerd`, `runc`, and `crun` all do some version of this.

Using `dockerd` as an example for this on the same host yields something like the following:

```sh
type=PROCTITLE msg=audit(06/22/26 16:34:15) : proctitle=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
type=SYSCALL msg=audit(06/22/26 16:34:15) : arch=x86_64 syscall=setns success=yes exit=0 a0=0x21
    a1=CLONE_NEWNET a2=0x0 a3=0x0 items=0 ppid=1 pid=800
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=(none) ses=unset comm=dockerd exec=/usr/bin/dockerd subj=unconfined key=container-config
```

There are a couple key differences here that are worth detecting on:

| Description | Regular | Escape |
| -- | -- | -- |
| The `setns` syscall received far more namespaces in the escape attempt | a1=CLONE_NEWNET | a1=CLONE_NEWNS\|CLONE_NEWUTS\|CLONE_NEWIPC\|CLONE_NEWPID\|CLONE_NEWNET |
| Auditd recorded a terminal in the escape attempt, but not in the daemon | tty=(none) | tty=pts0 |
| The nsenter command was used to perform the escape | comm=dockerd | comm=nsenter |

Monitoring each of these in isolation is a recipe for a bad time, but layering them helps avoid false positives.
The main one that is worth your attention is the `a1` syscall flags. Of the escape attempts I have seen,
none of them try to just use only the namespaces they need. Spinning up new containers also isolates the same resources, so
the process initiating this request is also key to detecting this attack in context.
Additionally, if you've read my [container-lab](/container-lab/) writeup series, you'll know that namespaces often need to be
paired to offer true isolation, and the reverse is true for breaking out of them.

### Open Source Rules

I'd be remiss not to mention there are some open source rules for detecting this behavior:

* Elastic's [Nsenter to PID Namespace via Auditd](https://github.com/elastic/detection-rules/blob/main/rules/linux/privilege_escalation_auditd_nsenter_target_host_pid.toml)
* Elastic's [Potential Docker Escape via Nsenter](https://github.com/elastic/detection-rules/blob/main/rules/linux/privilege_escalation_docker_escape_via_nsenter.toml)
* Elastic's [Potential Privilege Escalation via unshare and UID Change](https://github.com/elastic/detection-rules/blob/main/rules/linux/privilege_escalation_overlayfs_local_privesc.toml)
