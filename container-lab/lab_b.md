---
layout: series
title: Mount Namespace and Root Filesystem Isolation
permalink: /container-lab/lab-b/
series: Container Lab Walkthroughs
weight: 2.0
---

[https://github.com/molitorclay/container-lab/tree/main/lab/B](https://github.com/molitorclay/container-lab/tree/main/lab/B)

## Solution

```sh
$ unshare --mount --user --map-root-user bash -c "mount --bind -o ro ../flag_public ./flag_public && chroot . /bin/busybox sh ./bomb.sh 2>/dev/null"
```

## Step-by-Step Walkthrough

1. The quickest thing to try is just running the script.

    ```sh
    $ unshare bash -c ./bomb.sh
    ❌ Can't read /flag_public
    💥 BOOM!!! 💥
    ```

    The filesystem is not isolated yet, so its root doesn't align with what the script expects.

2. The quickest way to change the root filesystem is with a chroot jail.

    ```sh
    $ chroot . /bin/sh
    chroot: cannot change root directory to '.': Operation not permitted
    ```

    Unfortunately, we Need root permissions to chroot.

3. We can separate our permissions from the host using a User namespace though and ultimately elevate our permissions.

    ```sh
    $ unshare --user bash -c "chroot . /bin/bash -c ./bomb.sh"
    chroot: cannot change root directory to '.': Operation not permitted
    ```

    Without the `--map-root-user` flag we start as the `nobody` account in our container though, so we don't have appropriate permissions still.

4. We can become "root" in our new namespace without being actual root on the host by mapping a root in the namespace.

    ```sh
    $ unshare --user --map-root-user bash -c "chroot . /bin/bash -c ./bomb.sh"
    chroot: failed to run command ‘/bin/bash’: No such file or directory
    ```

    Even though we successfully chroot'ed, changing the rootfs to a directory without a "bin/" subdirectory means we don't have a `/bin/bash` file to call.

5. The scenario provides a handy `install-busybox.sh` script, which creates the `bin/` and `etc/` folders for our container.

    ```sh
    $ ./install-busybox.sh
    $ unshare --user --map-root-user bash -c "chroot . /bin/bash ./bomb.sh"
    chroot: failed to run command ‘/bin/bash’: No such file or directory
    ```

    Only, we still didn't install a `bash` binary in our new filesystem.

6. Since we have `bin/busybox` now, we can use that with the shell built-in to execute our script.

    ```sh
    $ unshare --user --map-root-user bash -c "chroot . /bin/busybox sh ./bomb.sh"
    ❌ Can't read /flag_public
    💥 BOOM!!! 💥
    ```

    We never solved the flag's path though. The chroot filesystem still doesn't have the flag file in it.

7. Now we can try to mount the `flag_public` inside the chroot.

    ```sh
    $ unshare --mount --user --map-root-user bash -c "mount --bind ../flag_public . && chroot . /bin/busybox sh ./bomb.sh"
    mount: /home/jerry/lab/B: wrong fs type, bad option, bad superblock on /home/jerry/lab/flag_public, missing codepage or helper program, or other error.
        dmesg(1) may have more information after failed mount system call.
    ```

    I tried to mount a file (`../flag_public`) onto a directory (`.`) instead of another file, which gave us the "wrong fs type" error.

8. Knowing that, we can properly mount the flag_public file on the host to a flag_public file inside our container.

    ```sh
    $ unshare --mount --user --map-root-user bash -c "mount --bind ../flag_public ./flag_public && chroot . /bin/busybox sh ./bomb.sh 2>/dev/null"
    mount: ./flag_public: mount point does not exist.
        dmesg(1) may have more information after failed mount system call.
    ```

    The destination doesn't exist, and `mount` does not make assumptions about what type it is. We can correct that.

9. Create an empty file with the `touch` command from our host, then access it in the container like the filesystem mount point that it is.

    ```sh
    $ touch ./flag_public
    $ unshare --mount --user --map-root-user bash -c "mount --bind ../flag_public ./flag_public && chroot . /bin/busybox sh ./bomb.sh"
    ✅ Can read /flag_public
    ❌ /flag_public is writable
    💥 BOOM!!! 💥
    ```

    We missed one last detail though: bind mounts are writable by default!

10. Mounting the file as read-only solves our last problem and defuses the bomb.

    ```sh
    $ unshare --mount --user --map-root-user bash -c "mount --bind -o ro ../flag_public ./flag_public && chroot . /bin/busybox sh ./bomb.sh 2>/dev/null"
    ✅ Can read /flag_public
    ✅ /flag_public is read-only
    ✅ flag_private is hidden
    🎉 You defused the bomb!!!
    ```

One of the final lessions: filesystem isolation requires a rootfs that looks like a host fs (`bin/` directory, `etc/` directory, etc.).
