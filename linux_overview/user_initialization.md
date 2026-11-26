---
layout: series
title: Linux User Initialization
permalink: /linux-overview/user-initialization/
series: Linux for Security Analysts
weight: 6.0
---

## Overview

* The init process
* Modern and Alternative init systems

## The init process

[https://tldp.org/HOWTO/HighQuality-Apps-HOWTO/boot.html](https://tldp.org/HOWTO/HighQuality-Apps-HOWTO/boot.html)

### System Initialization

### Responsibilities

### Runlevels

### Power Events

### Orphaned Processes

### Zombie Processes

### Startup Scripts

## Modern and Alternative init systems

### The Problems

Executing shell scripts in serial **SUCKS**. What if a script hangs? What if a script fails?

Ordering scripts by naming convention is also extremely tedious. Even in the best case, with just a few scripts, it is not fun.
With dozens or hundreds of scripts, it is near impossible to manage.

What happens if you need to insert a new script in the middle of execution?

[https://blog.darknedgy.net/technology/2020/05/02/0/](https://blog.darknedgy.net/technology/2020/05/02/0/)

### The Solution

> Executing shell scripts in serial **SUCKS**

Ok, let's execute them in parallel then.

> What if a script hangs?

Parallel scripts hanging wouldn't affect other scripts.

> What if a script fails?

Dependent scripts are (necessarily) affected, but nothing else is.

Modern init systems determine the best way to order scripts automatically.

Need a webserver? Wait for the network to be up. 

Have a backup system? Wait for the filesystem to show up to do backups.

#### systemd

#### Upstart
