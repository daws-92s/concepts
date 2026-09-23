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

**Why Linux over Windows for servers:**
- **Cost** — Linux is free and open source, no per-server license fee. Windows Server requires a paid license, which adds up fast across many servers.
- **Stability** — Linux servers can run for long stretches without needing a reboot, which matters for anything expected to stay up continuously.
- **Security** — a strict permissions model, faster community patching (since the source is open), and a smaller target for malware compared to Windows, which is more widely targeted simply because of how common it is.
- **Customization** — full access to the source and a huge range of distributions means you can strip an install down to exactly what a server needs, instead of carrying a general-purpose desktop OS's overhead.

This is why Linux ends up as the default choice for servers, cloud infrastructure, and embedded devices, even though Windows still dominates on the desktop.

**Firewalls on a Linux server (Security Groups):**
Once a Linux server is reachable on the network, a firewall decides what traffic is actually allowed to reach it. In AWS, a **Security Group (SG)** is a firewall applied directly to the server:
- **Ingress (inbound)** — traffic coming *into* the server
- **Egress (outbound)** — traffic going *out* from the server, usually left open by default
- CIDR notation defines scope: `0.0.0.0/0` means any IP on the internet; `122.183.36.3/32` means exactly one IP
- Security Groups are **stateful** — allow inbound traffic on a port, and the matching outbound response is automatically allowed back out, without a separate rule

**Connecting to a Linux server — SSH and key pairs:**
SSH (Secure Shell) gives you a terminal session on a remote Linux server, encrypted end-to-end. Instead of a password, it commonly uses a key pair:
- **Private key** — stays on your machine, never shared. Treat it like a password.
- **Public key** — safe to share; goes on any server you want to log into.

How it proves who you are: the server holds your public key and challenges any connecting client to prove it holds the matching private key. The private key itself never travels over the network — only proof that you hold it does.

Generating a key pair:
```
ssh-keygen -f joindevops
```
This produces `joindevops` (the private key — keep secret) and `joindevops.pub` (the public key — copy this onto a server).

## Common problems and how to solve them
Choosing a community distro saves license cost but means no vendor support line when production breaks — worth deciding deliberately based on how critical the system is, not just picking the free option by default.

A common SSH mix-up: confusing the private key file with the `.pub` public key file — the one *without* `.pub` must never be shared or uploaded anywhere; only the `.pub` file goes on a server.

A common firewall mistake is leaving a Security Group rule wide open (`0.0.0.0/0`) on a sensitive port like SSH (22), because it's the easiest thing to type when you just want something to work. Scope it down to the smallest set of IPs that actually need access once the setup is confirmed working.

## Key takeaways
- OS = Kernel + Shell + Applications. The kernel talks to hardware; the shell is how you interact with it; applications are what you actually use.
- Linux was written from scratch in 1991, inspired by Unix/Minix, not copied from them — and Git came 14 years later, for a separate reason (kernel development tooling), not as part of the original release.
- "Distribution" = Linux kernel + a chosen set of tools and defaults, packaged for installation. Different distros trade off support (enterprise vs community) and footprint (full desktop vs embedded).
- Linux wins for servers on cost (free/open source), stability (long uptimes), security (permissions + fast community patching), and customization (strip it down to only what's needed).
- Mac bundles hardware and OS together; a standard PC lets you choose your OS — this is a deliberate trade-off between simplicity/control (Apple) and flexibility (PC).
- A Security Group is a stateful, per-server firewall — allow a port in, and the response traffic is automatically allowed back out. Default to the narrowest CIDR scope that works, not `0.0.0.0/0`.
- SSH key auth works because the private key never leaves your machine — the server only ever sees (and trusts) the public key. `ssh-keygen -f <name>` generates the pair; only `.pub` ever gets shared.

See also: [02-why-cloud-migration.md](02-why-cloud-migration.md)
