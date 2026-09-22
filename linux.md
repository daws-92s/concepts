# Operating Systems, Linux History, and Windows vs Linux

## What this is / why it matters
Every computer needs software to manage its hardware and run applications — that's the OS. Understanding how an OS is structured, where Linux came from, and why it dominates servers (while barely registering on desktops) explains a lot about why the tech industry is built the way it is: why cloud servers run Linux, why "flavors" of Linux exist, and why Windows and Linux serve different purposes.

## How it works

**Hardware vs software:**
- CPU, RAM, and storage are **hardware**
- The OS is **software** that sits on top and manages that hardware

**OS structure — three layers:**
- **Kernel** — the core that talks directly to hardware
- **Shell** — the interface (command line or GUI) that lets you interact with the kernel
- **Applications** — the programs you actually run

Kernel + Shell + Applications together make up a full OS.

**Hardware/software coupling — two models:**
- **Mac** — Apple sells you the hardware and the OS (macOS) as one tightly coupled package. You don't choose a different OS for a MacBook.
- **PC (e.g., Dell)** — you buy the hardware, and you're free to install whichever OS you want (Windows, Linux, etc.).

**Linux's origin:**
- 1991: Linus Torvalds, a student, started writing a free, Unix-style kernel from scratch in C, while working on a Unix-like teaching OS called Minix. He wasn't copying Minix's code — he was inspired by it and aimed to be Unix-compatible.
- Unix itself was an existing OS, but it was commercial and expensive to license — part of the motivation for a free alternative.
- Git (the version control tool) came much later — 2005 — built by Torvalds to manage Linux kernel development after a licensing dispute with the tool they'd been using. It wasn't part of the original 1991 release.

**Distributions ("distros"):**
Linux itself is just the kernel. A "distribution" packages the kernel with a shell, applications, and defaults into something installable — Red Hat, Ubuntu, SUSE, Amazon Linux, CentOS, AlmaLinux are all examples. Red Hat–family distros (RHEL, CentOS, AlmaLinux, Amazon Linux) share a common lineage and package format.
- **Enterprise distros** (like RHEL) come with paid, immediate vendor support — important if you need guaranteed response times for production issues.
- **Community distros** (like plain CentOS/Ubuntu) are free but you're on your own (or relying on community forums) when something breaks.

Linux also scales down a lot: a full desktop-style distro can be a couple of GB, while an embedded Linux build (for routers, IoT devices, etc.) can be as small as ~10MB.

**Windows vs Linux:**
- On desktops, Windows dominates (roughly 60%+ market share); Linux is a small minority (~3%).
- On servers, the picture flips — Linux runs the majority of servers and the vast majority of top websites, and it's effectively universal on supercomputers.
- Android phones run on a Linux kernel too, which is part of why Linux's real-world reach is much bigger than desktop numbers alone suggest.

## Common problems and how to solve them
choosing a community distro saves license cost but means no vendor support line when production breaks — worth deciding deliberately based on how critical the system is, not just picking the free option by default.

## Key takeaways
- OS = Kernel + Shell + Applications. The kernel talks to hardware; the shell is how you interact with it; applications are what you actually use.
- Linux was written from scratch in 1991, inspired by Unix/Minix, not copied from them — and Git came 14 years later, for a separate reason (kernel development tooling), not as part of the original release.
- "Distribution" = Linux kernel + a chosen set of tools and defaults, packaged for installation. Different distros trade off support (enterprise vs community) and footprint (full desktop vs embedded).
- Linux dominates servers and infrastructure, not desktops — know which one you're talking about before quoting a market-share number.
- Mac bundles hardware and OS together; a standard PC lets you choose your OS — this is a deliberate trade-off between simplicity/control (Apple) and flexibility (PC).

See also: [[why-cloud-migration]]
