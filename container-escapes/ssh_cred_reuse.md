---
layout: series
title: SSH Credential Reuse
permalink: /container-escapes/ssh-cred-reuse
series: Container Escapes
weight: 2.0
---

## Escaping the Container

Testing this involves creating an SSH key on the host, authorizing it for SSH access to the user, and sharing it with the container.
The key doesn't need to be created on the host, but I just did it for convenience.
An attacker pilfering it from a compromised developer workstation could also represent the same activity,
so long as the key is authorized for the host urnning the container we're escaping from.

```sh
# first prep the host by allowing an SSH key to log in
$ cat /home/ubuntu/.ssh/id_rsa.pub >> /home/ubuntu/.ssh/authorized_keys
# spawn a container with the SSH key from the host mounted as a volume
# note the container's username is also 'ubuntu', just like the host's
$ docker run --rm -it -v ~/.ssh:/home/ubuntu/.ssh ubuntu
# ssh back into the host via the shared SSH key
root@b264b5ae0319:/$ su ubuntu
ubuntu@b264b5ae0319:/$ cd ~
ubuntu@b264b5ae0319:~$ ssh ubuntu@172.17.0.1
```

Once the SSH session on the host has been opened, we can check out the new user login session from the host:

```sh
$ systemd-cgls -u user-1000.slice
Unit user-1000.slice (/user.slice/user-1000.slice):
|--user@1000.service
    ...<removed for brevity>...
|--session-64.scope
|   |--33693 sshd: ubuntu [priv]
|   |--33748 sshd: ubuntu@pts/4
|   |--33749 -bash
|   |--33800 systemd-cgls -u user-1000.slice
|   |--33801 pager
```

We can also consult the SSH authentication logs at `/var/log/auth.log` to see this in action (more below).

## Detection

Similar to [socket abuse](/container-escapes/docker-sockets), this authentication and breakout activity occurs
in the context of a system service - SSH instead of Docker in this instance.
We can use additional process telemetry, such as cgroups, to help us identify if the action's source is a container,
but the container processes themselves don't actually exhibit all that unusual behavior.

Like Docker, this is also due to the SSH daemon receiving a legitimate (from its perspective) request to open a new SSH session.
The new session will then look like any other user login session, except for its origin connection.

As far as detection goes, reviewing SSH authentication logs in addition to Auditd telemetry should be a high-confidence source of identifying escapes.
Successful connections from the Docker bridge IP (`172.17.0.2`) can be a good indicator, but not every container runtime uses the same bridge network.
Additionally, we can tell the SSH request came from a namespaced user (`ubuntu(uid=0)` instead of `uid=1000`),
but this would be based on the typical container image definition.
This type of information would be more useful during forensic analysis after an incident has already been initiated.

### How It Works

Below is a snapshot of the SSH authentication logs showing a connection from a local container to the SSH daemon:

```sh
2026-06-19 HOSTNAME sshd[33693]: Accepted publickey for ubuntu from 172.17.0.2 port 53452 ssh2: RSA SHA256://<blah>
2026-06-19 HOSTNAME sshd[33693]: debug1: monitor_child_preauth: user ubuntu authenticated by privileged process
2026-06-19 HOSTNAME sshd[33693]: debug1: auth_activate_options: setting new authentication options [preauth]
2026-06-19 HOSTNAME sshd[33693]: debug1: monitor_read_log: child log fd closed
2026-06-19 HOSTNAME sshd[33693]: debug1: PAM: establishing credentials
2026-06-19 HOSTNAME sshd[33693]: pam_unix(sshd:session): session opened for user ubuntu(uid=1000) by ubuntu(uid=0)
```

One important attribute of this login session is displayed on the first line: `Accepted publickey for ubuntu from 172.17.0.2`.
This gives us the source IP for the connection attempt, which translates to our Docker container's IP.

The last line is also interesting here: `session opened for user ubuntu(uid=1000) by ubuntu(uid=0)`
since I am logging into my host `ubuntu` account (uid=1000) by the container `ubuntu` account (uid=0), SSH records bother usernames.
It might make more sense to read this as the following:

```sh
2026-06-19 HOSTNAME sshd[33693]: pam_unix(sshd:session): session opened for user host-user(uid=1000) by contained-root(uid=0)
```

Additionally, just like our host volume mounts, the SSH process records the uid of the contained user *from the container's perspective*.
The contained user is not actually running as the root user on the host, but SSH records it as uid=0 anyway.

### Open Source Rules

From an endpoint perspective, this is difficult to detect. From an application perspective, we'll have an easier time.
