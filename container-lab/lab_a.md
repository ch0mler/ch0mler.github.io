---
layout: page
title: "Container Lab A: PID and UTS Namespaces"
permalink: /container-lab/lab-a/
---

[https://github.com/molitorclay/container-lab/tree/main/lab/A](https://github.com/molitorclay/container-lab/tree/main/lab/A)

## Solution

```sh
$ unshare --mount --pid --fork --user --map-root-user --uts bash -c "hostname jerry-pc && mount -t proc proc /proc && ./bomb.sh"
```

## Step-by-Step Walkthrough

1. First thing, let's try to change the hostname directly.

    ```sh
    $ hostname jerry-pc
    hostname: you must be root to change the host name
    ```

    We don't have root though, so we quickly encounter a permissions error.

2. The `uts` namespace separates hostnames, so let's set up our own namespace.

    ```sh
    $ unshare --uts bash
    unshare: unshare failed: Operation not permitted
    ```

    ...But we're still not the root user on this host, so we have no control of namespaces.

3. We can add a user namespace which wouldn't restrict our permissions to the unshare call.

    ```sh
    $ unshare --user --uts bash -c "whoami && hostname jerry-pc"
    nobody
    hostname: you must be root to change the host name
    ```

    Now we have a separate list of users, but we're still not the root user
    inside our new namespace. We still need to elevate permissions to change host name.

4. We can become "root" in our new namespace without being actual root on the host by mapping a root in the namespace.

    ```sh
    $ unshare --user --map-root-user --uts bash -c "whoami && hostname jerry-pc && ./bomb.sh"
    root
    ✅ Running on jerry-pc
    ❌ Process count out of range: 23
    💥 BOOM!!! 💥
    ```

    Excellent, we've made progress. We are root in the new namespace,
    but now we have too many processes for the bomb to defuse.

5. We can keep adding to the list of namespaces by unsharing the PID namespace.

    ```sh
    $ unshare --pid --user --map-root-user --uts bash -c "whoami && hostname jerry-pc && ./bomb.sh"
    root
    bash: fork: Cannot allocate memory
    ```

    The [unshare manual page](https://www.man7.org/linux/man-pages/man1/unshare.1.html) indicates persistent PID namespaces won't work without also forking the shell into a child of unshare instead of executing it directly.

6. Let's add `--fork` to spawn our shell and its children as subprocesses.

    ```sh
    $ unshare --pid --fork --user --map-root-user --uts bash -c "whoami && hostname jerry-pc && ./bomb.sh"
    root
    ✅ Running on jerry-pc
    fatal library error, lookup self
    ❌ Process count out of range: 0
    💥 BOOM!!! 💥
    ```

    We're very close. Examining `/proc/$PID/self` leads to a fatal error from `ps ax --no-headers | wc -l` in the bomb script though.

7. In order for `/proc/$PID/self` to be available to the bomb script, we have to mount the `/proc` directory as a procfs type.

    ```sh
    $ unshare --pid --fork --user --map-root-user --uts bash -c "whoami && hostname jerry-pc && mount -t proc proc /proc && ./bomb.sh"
    root
    mount: /proc: permission denied.
          dmesg(1) may have more information after failed mount system call.
    ```

    Similary to before, mounting over `/proc` requires root privileges on the host, and we only have "root" privileges in the container.

8. Adding a fourth namespace, `mount`, to our list of existing namespaces in the container should give us the final piece.

    ```sh
    $ unshare --mount --pid --fork --user --map-root-user --uts bash -c "whoami && hostname jerry-pc && mount -t proc proc /proc && ./bomb.sh"
    root
    ✅ Running on jerry-pc
    ✅ Number of processes: 4
    🎉 You defused the bomb!!!
    ```

    And it worked!
