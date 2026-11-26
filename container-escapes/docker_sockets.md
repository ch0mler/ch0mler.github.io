---
layout: series
title: Docker Socket Abuse
permalink: /container-escapes/docker-sockets
series: Container Escapes
weight: 2.0
---

## Escaping the Container

Testing this is fairly straightforward: mount the `docker.sock` file, or an equivalent depending on runtime, into your container.
I've only shown two possible ways to do this, but there are as many methods for doing this as there are ways to spawn an HTTP request.
Using the `docker` binary is easy *if it's already installed in the container*.
I'm installing it myself in the container, but that's honestly a pretty sketchy thing to do in something that's supposed to be immutable.

If we were going to try to be more exhaustive, we could also use Python, Ruby, Go, a custom binary, or shell built-ins to perform this as well.
This type of attack really shows how brittle reliance on command lines alone can be.
In order to detect this reliably, we'll want to analyze the logs coming off our container runtime (Docker) or
container orchestration layer (Kubernetes) - the thing actually responsible for interpreting and actioning these requests.

```sh
# start a container with the Docker socket available
$ docker run -it -v /var/run/docker.sock:/var/run/docker.sock --rm ubuntu:latest /bin/bash
# install the docker binary and spawn a new container
root@c9d12c7e246b:/$ apt update && apt install -y docker.io
root@c9d12c7e246b:/$ docker run alpine:latest
# or use curl to interact directly with the Docker API
root@c9d12c7e246b:/$ curl -s --unix-socket /var/run/docker.sock -X POST -H "Content-Type: application/json" \
    -d '{"Image": "alpine", "Cmd": ["sleep", "1000"], "AttachStdin": false, "AttachStdout": false, "AttachStderr": false, "Tty": false, "OpenStdin": false}' \
    http://localhost/containers/create?name=breakout | jq -r '.Id' > /container_id.txt
# we've created the container,  but we also have to run it now
root@c9d12c7e246b:/$ curl -s --unix-socket /var/run/docker.sock -X POST http://localhost/containers/$(cat /container_id.txt)/start
```

## Detection

### How It Works

There are quite a few more logs to sift through with these types of escapes, mainly due to
the use of multiple distinct processes to setup and prepare the new containers being launched.

From a pure Auditd perspective, this can be a really difficult attack to identify:
command lines are brittle, the Docker daemon is doing what it usually does, and there is nothing unusual in the container process lineage.
The best place to focus on is application logs for your container service - in my case Docker.
The same ideas apply to privilege escalation techniques in Podman, LXC, and Kubernetes though -
use the application logs to identify where requests are coming from, and examine the contents of the requests to understand what is being spawned.

#### Using Docker

A snippet of the audit logs with the most relevant entries can be found below.

What most commands my attention is the sub-second timespan all this activity happens in.
Everything happens in the same second (`21:04:33`) and can give us a tight time window to narrow our detections to.
If we collected something like cgroups along with process executions, we could also tie the `docker run` command to a particular container via the cgroup path.
This would give us another layer to build detections off of. With just this audit logging enabled though, we don't have a lot other than typical container initialization.

```sh
# the initiating docker process running inside the container
type=PROCTITLE msg=audit(06/19/26 21:04:33) : proctitle=docker run alpine:latest
type=PATH msg=audit(06/19/26 21:04:33) : item=1 name=/lib64/ld-linux-x86-64.so.2 inode=519849 dev=00:27 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/19/26 21:04:33) : cwd=/
type=EXECVE msg=audit(06/19/26 21:04:33) : argc=3 a0=docker a1=run a2=alpine:latest
type=SYSCALL msg=audit(06/19/26 21:04:33) : arch=x86_64 syscall=execve success=yes exit=0 a0=0x5c9a4ec490b0 a1=0x5c9a4ec69770 a2=0x5c9a4ec48ed0 a3=0x31
    items=2 ppid=12498 pid=16601 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=docker exec=/usr/bin/docker subj=docker-default key=process_creation
type=SYSCALL msg=audit(06/19/26 21:04:33) arch=x86_64 syscall=connect success=yes exit=0 a0=0x3 a1=0xc002cd710 a2=0x17 a3=0x0 items=1
    ppid=12498 pid=16601 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset cmom=docker exe=/usr/bin/docker subj=docker-default key=external-access
# ... <snip> ...
# the containerd service receiving the request via the Docker socket
type=PROCTITLE msg=audit(06/19/26 21:04:33) : proctitle=/usr/bin/containerd-shim-runc-v2 -namespace moby -address /run/containerd/containerd.sock -publish-binary /usr/bin/containerd -i
type=PATH msg=audit(06/19/26 21:04:33) : item=0 name=/usr/bin/containerd-shim-runc-v2 inode=1062 device=ca:01 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/19/26 21:04:33) : cwd=/run/containerd/io.containerd.runtime.v2.task/moby/c9d12c7e246b
type=EXECVE msg=audit(06/19/26 21:04:33) : argc=10 a0=/usr/bin/containerd-shim-runc-v2 a1=-namespace a2=moby a3=-address a4=/run/containerd/containerd.sock
    a5=-publish-binary a6=/usr/bin/containerd a7=id a8=c9d12c7e246b a9=start
type=SYSCALL msg=audit(06/19/26 21:04:33) : arch=x86_64 syscall=execve success=yes exit=0 a0=0xc0002cfe90 a1=0xc000e2eb40 a2=0xc000e24c00 a3=0x0
    items=1 ppid=609 pid=16609 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=(none) ses=unset comm=containerd-shim exe=/usr/bin/containerd-shim-runc-v2 subj=unconfined key=process_creation
# ... <snip> ...
# the runc container runtime actually setting up the new container
type=PROCTITLE msg=audit(06/19/26 21:04:33) : proctitle=runc init
type=SYSCALL msg=audit(06/19/26 21:04:33) : arch=x86_64 syscall=unshare success=yes exit=0
    a0=CLONE_NEWNS|CLONE_STOPPED|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWPID|CLONE_NEWNET a1=0x5ac38097458b a2=0x5 a3=0x0 items=0 ppid=16629 pid=16641
    auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root tty=(none) ses=unset
    comm=runc:[1:CHILD] exec=/runc subj=unconfined key=container-config
# ... <snip> ...
# the runc container runtime setting the namespaces on the new container
type=PROCTITLE msg=audit(06/19/26 21:04:33) : proctitle=runc --root /var/run/docker/runtime-runc/moby --log /run/containerd/io.containerd.runtime.v2.task/moby/c9d12c7e246b
type=PATH msg=audit(06/19/26 21:04:33) : item=0 name=/ inode=2 dev=ca:01 mode=dir,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(06/19/26 21:04:33) : cwd=/run/containerd/io.containerd.runtime.v2.task/moby/c9d12c7e246b
type=SYSCALL msg=audit(06/19/26 21:04:33) : arch=x86_64 syscall=setns success=yes exit=0 a0=0x10 a1=CLONE_NEWNS a2=0x0 a3=0x0
    items=1 ppid=16618 pid=16629 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=(none) ses=unset comm=runc exe=/usr/bin/runc subj=unconfined key=container-config
```

**Note**: The container ID of `c9d12c7e246b` was shortened in these audit logs for readability.

#### Using curl

A snippet of the audit logs with the most relevant entries can be found below.

Similar to the docker commands above, we could do better with our detections.
However, it is quite unusual in most environments to interact with the docker socket file using `curl`.
There are always exceptions to this, and it can be quite brittle because we're relying on command line arguments specific to a particular binary,
but focusing on `curl` or `wget` issuing JSON requests over HTTP to this socket should typically be a strong signal that we need to investigate further.

```sh
# setting up the container spec
type=PROCTITLE msg=audit(06/19/26 21:14:51) : proctitle=curl -s --unix-socket /var/run/docker.sock -X POST -H Content-Type: application/json -d {\012"Image": "alpine",
type=PATH msg=audit(06/19/26 21:14:51) : item=1 name=/lib64/ld-linux-x86-64.so.2 inode=519849 dev-00:27 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(06/19/26 21:14:51) : item=0 name=/usr/bin/curl inode=531348 dev=00:27 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fer=0 cap_frootid=0
type=CWD msg=audit(06/19/26 21:14:51) : cwd=/
type=EXECVE msg=audit(06/19/26 21:14:51) : argc=11 a0=curl a1=-s a2=--unix-socket a3=/var/run/docker.sock a4=-X a5=POST a6=-H a7=Content-Type: application/json a8=-d
    a9={\012"Image": "alpine",\012"Cmd": ["sleep", "1000"],\012"AttachStdin": false,\012"AttachStdout": false,\012"AttachStderr": false,\012"Tty": false,\012"OpenStdin": false}
    a10=http://localhost/containers/create?name=breakout
type=SYSCALL msg=audit(06/19/26 21:14:51) : arch=x86_64 syscall=execve success=yes exit=0 a0=0x5c9a4ec6b280 a1=0x5c9a4ec69c30 a2=0x5c9a4ec48ed0 a3=0x8
    items=2 ppid=12498 pid=17000 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=curl exe=/usr/bin/curl subj=docker-default key=process_creation
# ... <snip> ...
# actually running the container
type=PROCTITLE msg=audit(06/19/26 21:15:06) : proctitle=curl -s --unix-socket /var/run/docker.sock -X POST http://localhost/containers/b3c781aecce8/start
type=PATH msg=audit(06/19/26 21:15:06) : item=1 name=/lib64/ld-linux-x86-64.so.2 inode=519849 dev-00:27 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(06/19/26 21:15:06) : item=0 name=/usr/bin/curl inode=531348 dev=00:27 mode=file,755
    ouid=root ogid=root rdev=00:00 nametype=NORMAL cap_fp=none cap_fi=none cap_fe=0 cap_fer=0 cap_frootid=0
type=CWD msg=audit(06/19/26 21:15:06) : cwd=/
type=EXECVE msg=audit(06/19/26 21:15:06) : argc=7 a0=curl a1=-s a2=--unix-socket a3=/var/run/docker.sock a4=-X
    a5=POST a6=http://localhost/containers/b3c781aecce8/start
type=SYSCALL msg=audit(06/19/26 21:14:51) : arch=x86_64 syscall=execve success=yes exit=0 a0=0x5c9a4ec4bf80 a1=0x5c9a4ec46f40 a2=0x5c9a5ec48ed0 a3=0x81
    items=2 ppid=12498 pid=17007 auid=unset uid=root gid=root euid=root suid=root fsuid=root egid=root sgid=root fsgid=root
    tty=pts0 ses=unset comm=curl exe=/usr/bin/curl subj=docker-default key=process_creation
```

**Note**: The container ID of `b3c781aecce8` was shortened in these audit logs for readability.

### Open Source Rules

Elastic has posted some detection rules for monitoring Docker socket abuse, though one of them is a little brittle given it relies on command line argument matching.

* Elastic's [Potential Privilege Escalation through Writable Docker Socket](https://github.com/elastic/detection-rules/blob/main/rules/linux/privilege_escalation_writable_docker_socket.toml)
* Elastic's [Unusual Process Connection to Docker or Containerd Socket](https://github.com/elastic/detection-rules/blob/main/rules/linux/discovery_unusual_process_connection_to_container_runtime_socket.toml)
