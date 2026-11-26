---
layout: post
title: Detecting Container Escapes
permalink: /posts/container-escapes
tags: [containers]
---

Not too long ago, I looked into how different classes of container escapes worked, and
how to detect them using the auditd framework.
This series of posts covers the 4 main escape types I looked into:
namespace modifications, SSH credential reuse, Docker socket abuse, and host volume mounts.

[Container Escape Detections](/container-escapes/)

While researching this, I also found a very in-depth walkthrough on building an
Auditd ruleset on Medium by a user called IzyKnows:

[https://izyknows.medium.com/linux-auditd-for-threat-detection-d06c8b941505](https://izyknows.medium.com/linux-auditd-for-threat-detection-d06c8b941505)
