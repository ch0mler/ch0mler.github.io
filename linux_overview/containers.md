---
layout: page
title: "7. Containers"
permalink: /linux-overview/containers/
---

## Overview

* Container Basics
* How do containers work?
* Container Technologies
* Practical Examples

## Container Basics

While containers are incredibly common in production deployments these days,
you may still be asking why are they so popular.

What about containers is different from directly deploying applications to servers?

What benefits do containers give us over a process like virtualization?

The hallmarks of containers are:

1. **Lightweight** - they are limited to just enough resources to function
2. **Portable** - thanks to apps like Docker, container setup is independent of the host
3. **Isolated** - resources are limited and can be logically grouped together using Linux kernel features

## How do containers work?

The Linux kernel has many different tools for isolating processes,
and containers simply wrap them all up with a pretty bow.

Notably, whenever you run a tool like Rancher or Docker Desktop,
you are likely running your containers from a Linux VM rather than on the host.
This is because other operating systems don't have the same tools for isolating processes that Linux does, and it is easier to just ship with a VM rather than re-implementing these features in other kernels.

### Control Groups (cgroups)

A cgroup is a collection of processes that are labeled and bound by the same resource limits.
These resources can be CPU time, memory, input/output, and the number of concurrent PIDs.
Since containers are isolated, restricting the amount of resources available to them 
is a core function - a single container should not be able to exhaust a particular 
resource and have it affect every other container on the host.

### Namespaces

Much like cgroups, a namespace labels a set of processes, which then have restricted visibility
into specific resources, such as processes or network interfaces. These specific resource namespaces include:

* Process (PID)
* Mount
* User
* Network
* Hostname
* Inter-process communication (IPC)
* Control Groups

Since container isolation also requires reducing visibility across containers, namespaces are the go-to method for doing so.
In most cases, multiple types of restrictions must be applied to comprehensively isolate two processes.

See my [container-lab](/container-lab/) walkthroughs for step-by-step views into why that is the case.

### Chroot

Literally "changing the root", `chroot` is a program that runs a process with a modified view of the root filesystem.
This was used for many, many years before containers were a thing, and it spawned the term "chroot jail".

Containers that have been isolated via namespace, but which still have total access to the filesystem,
can break their isolation boundaries by reading sensitive information about the host from `/proc`,
or by writing to locations outside their purview.
Since [everything is a file](/linux-overview/linux-filesystem/), this is highly dangerous for system integrity.

## Container Technologies

Even though we know what containers are in general, it can still be helpful to look at a few examples of them in practice.

### Docker

The most widely known container configuration tool.
This was probably the single application that brought containers into the mainstream, because it made creating and distributing them significantly easier.

In addition to a file format, which has since been handed over to the [CNCF](https://www.cncf.io/), Docker is also a service and build environment. The Docker ecosystem addresses few different use cases that other projects do, like `runc` (a container runtime), Podman (a build interface and service), and Kubernetes (a container orchestrator).

[https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)

### Kubernetes

The go-to container orchestrator.
If you suddenly get 3x site traffic compared to normal, Kubernetes is intelligent 
enough to know it should spin up more containers and keep them all behind a load 
balancer...well, if it's configured properly.

[https://kubernetes.io/docs/concepts/overview/#why-you-need-kubernetes-and-what-can-it-do](https://kubernetes.io/docs/concepts/overview/#why-you-need-kubernetes-and-what-can-it-do)

### Ubuntu Snap

Pre-packaged applications, such as Steam or PowerShell, used to make building and deploying applications easier. Next year is always the year of Desktop on Linux!

[https://snapcraft.io/about](https://snapcraft.io/about)

## Practical Examples

### A Normal Docker Application

1. User issues command with the `docker` command line tool
2. The `docker` CLI tool requests the `containerd` service to set up the container
3. The `containerd` service parses Docker commands, pulls images, and executes the `runc` runtime to actually create the containerized process with appropriate isolation
4. The `runc` runtime performs low-level actions to create containerized processes according to the user's specifications

### A Generic Container Escape

We'll visualize a container escape using a generic escape sequence going through the Docker socket, such as one [outlined here by Spark42](https://blog.spark42.tech/breaking-the-chains-two-ways-to-escape-a-docker-container/).

1. User issues command with the `docker` command line tool
2. The `docker` CLI tool requests the `containerd` serivce to set up the container
3. The `containerd` service parses Docker commands, pulls images, and executes the `runc` runtime to actually create the containerized process with appropriate isolation
4. The `runc` runtime performs low-level actions to create containerized processes according to the user's specifications
5. The containerized process issues arbitrary commands back to the `containerd` service via the Docker socket
6. `containerd` interprets the Docker commands and spawns new `runc` processes to set up additional, malicious containers with different isolation boundaries.

## Sources

* [Container Security by Liz Rice](https://www.oreilly.com/library/view/container-security/9781492056690/)
* [cgroups(7) manual](https://www.man7.org/linux/man-pages/man7/cgroups.7.html)
* [The container is a lie!](https://upsun.com/blog/the-container-is-a-lie/)
* [cgroupv2](https://chrisdown.name/talks/cgroupv2/cgroupv2-fosdem.pdf)
* [Audit, namespaces, and containers](https://lwn.net/Articles/699819/)

---

<footer>
  <br>
  <a href="/linux-overview/">Back to Linux for Security Analysts</a>
</footer>
