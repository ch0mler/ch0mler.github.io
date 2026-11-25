---
layout: page
title: "1. The UNIX Philosophy"
permalink: /linux-overview/unix-philosophy/
---

What is the UNIX Philosophy, and why should we care?

## A Quick History

[https://en.wikipedia.org/wiki/Unix_philosophy](https://en.wikipedia.org/wiki/Unix_philosophy)

Back in the late 1960's, the UNIX OS was created at Bell Labs.
Ken Thompson was unsatisfied with available operating systems, so he decided to make a new one.
A few years in, Ken and Dennis Ritchie, another scientist at Bell Labs, published a paper with guidelines
on building software, which they carried forward into their new OS. 

## The UNIX Philosophy

Those guidelines went roughly like this:

1. **Make each program do one thing well.** To do a new job, build afresh rather than complicate old programs by adding new features.
2. Expect the **output of every program to become the input to another**, as yet unknown, program.
    Don't clutter output with extraneous infromation. Avoid stringently columnar or binary input formats. Don't insist on interactive input.
3. **Design and build software**, even operating systems, **to be tried early**, ideally within weeks.
    Don't hesitate to throw away the clumsy parts and rebuild them.
4. **Use tools in preference to unskilled help** to lighten a programming task, even if you have to detour
    to build the tools and expect to throw some of them out after you've finished using them

[https://archive.org/details/UNIX-Time-Sharing-System](https://archive.org/details/UNIX-Time-Sharing-System)

---

A few years later, Dough McIlroy shortened these guidelines to the following (more widely known) maxims:

1. Write programs that **do one thing** and do it well
2. Write programs to **work together**
3. Write programs to **handle text streams**, because that is a universal interface

[https://cscie2x.dce.harvard.edu/hw/ch01s06.html](https://cscie2x.dce.harvard.edu/hw/ch01s06.html)

Doug is the inventor of the UNIX pipe, which is going to be exceedingly interesting to us when analyzing security events for relationships between processes.

## Why does it matter?

Modern Unix-derived operating systems, including the Linux kernel and macOS (BSD-based),
still encourage users to develop according to these guidelines.

Since software engineers, scientists, and researchers often use these types of OSes in their daily tasks,
and Linux is easily the most dominant kernel powering servers on the Internet, understanding how the tools on these hosts work helps us differentiate between normal and abnormal behavior.

Breaking down each guideline separately, we can see:

1. Write programs that **do one thing** and do it well
    * Single-purpose tools can make intent much more obvious as long we can identify how the tools are connected together
2. Write programs to **worth together**
    * Sister processes are at least as important for building endpoint context as parent/child relationships are
3. Write programs to **handle text streams**, because that is a universal interface
    * Text processing tools rule Linux, so we will likely see attackers use the same binaries that administrators do

## Conclusion

Hopefully this post adequately explains how the ideas that built UNIX decades ago are also relevant to modern cybersecurity folks.
If you are responsible for protecting production servers or developer workstations, this information
should help give you a few higher-level ideas to keep in mind while analyzing security logs or building detections.

---

<footer>
  <br>
  <a href="/linux-overview/">Back to Linux for Security Analysts</a>
</footer>
