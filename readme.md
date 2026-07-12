# Linux/Sysadmin Fundamentals - Self-Study Syllabus

## Who this is for
You're an intermediate Linux user - a full-stack developer moving into DevOps/SRE. You're already comfortable with basic commands and the terminal; this isn't a beginner course. Every resource below was chosen against one bar - build intuition with an analogy first, then walk through the real underlying mechanism, and never sacrifice accuracy for simplicity. Wherever possible the pick is pictorial/drawn (Julia Evans' comics, Core Dumped's animations), because that's how this material sticks best for you.

This syllabus is the trained-down output of the research in [main-plan.md](main-plan.md) - that file has the full reasoning, quotes, and runner-up resources for the original 17 topics. The 9 topics added later (containers, DNS, OOM killer, LVM/RAID, package management, iptables/nftables, SELinux/AppArmor) were researched the same way, directly against the same NetworkChuck bar - sources are cited inline in each module rather than in a separate research doc.

## How to use this
- Work through the 6 modules **in order** - each one leans on concepts from the last (you need processes before systemd makes sense, you need TCP before TLS makes sense, you need namespaces+cgroups before containers make sense).
- Within a module, topics are also ordered - don't skip around on a first pass.
- For each topic: consume the resource, then do the **Confirm it** exercise. It's deliberately short - a single command or observation, not a project. If you can predict the output before you run it, you've actually learned the mechanism, not just watched something about it.
- At the end of every module there's a **🧪 capstone** - a slightly bigger task on a disposable VM (a cheap cloud box, or a local VM via multipass/VirtualBox/Vagrant - anything you can safely destroy) that makes you use everything in the module together, not in isolation. This is where the knowledge actually locks in. Destroy the VM when you're done with it; the point was the observation, not the box.
- Tick the `- [ ] ` boxes in each module file as you go. It's your own file - edit it freely.
- Keep **Brian Ward's *How Linux Works, 3rd ed.* (No Starch, 2021)** on hand as a running companion the whole way through. It's not tied to one topic - it's the connective-tissue reference for processes, signals, boot, filesystems, and the privilege model, and it's explicitly the safety net for the topics flagged below.

https://stcformation.com/wp-content/uploads/2023/10/How-Linux-Works-What-Every-Superuser-Should-Know.pdf

## ⚠️ Topics with no fully-matching resource
These topics didn't have a single resource hitting analogy + mechanism + visual at the bar you set. They're flagged inline in their module with the best available substitute (usually text/prose, not illustrated) - go in expecting that, not expecting a Julia-Evans-quality comic:
- Signals (Module 1)
- Environment variables & shell inheritance (Module 1)
- Disk/inode exhaustion (Module 2)
- LVM/RAID - partially flagged, a whiteboard-video exists but not a comic (Module 2)
- Package management internals (Module 2)
- iptables/nftables - partially flagged, the network-namespace half is well illustrated, the packet-filtering half isn't (Module 3)
- sudo vs su (Module 4)
- SELinux vs AppArmor (Module 4)
- Cron internals - partially flagged (Module 6)

Everything else - including all three container topics and DNS - turned out to have a genuinely illustrated match (mostly more Julia Evans zines than expected), so they're not flagged.

## The 6 modules

| # | Module | Topics | Builds on |
|---|--------|--------|-----------|
| 1 | [Process & OS Fundamentals](module-1-process-fundamentals.md) | Processes/fork-exec/zombies, file descriptors, permissions, signals, the OOM killer & memory/swap, env vars & shell inheritance | Nothing - start here |
| 2 | [Filesystem & Boot](module-2-filesystem-boot.md) | Inodes/hard links/symlinks, LVM/RAID, FHS, disk/inode exhaustion, package management (apt/dpkg), systemd units, boot process | Module 1 (processes, permissions) |
| 3 | [Networking](module-3-networking.md) | TCP/IP handshake & sockets, DNS, iptables/nftables & network namespaces, reverse proxy internals, TLS/HTTPS handshake, SSH key exchange | Module 1 (processes, fds) |
| 4 | [Authentication & Privilege](module-4-auth-privilege.md) | PAM, sudo vs su, SELinux vs AppArmor | Module 1 (permissions/setuid) |
| 5 | [Containers & Modern Infra](module-5-containers.md) | Linux namespaces, cgroups, container runtime internals | Modules 1, 3 (processes, network namespaces) - the DevOps/SRE payoff module |
| 6 | [Automation](module-6-automation.md) | Cron internals & race conditions | Modules 1–5 (closing module) |

## Why this order (not the original 1–17 numbering, and why 9 topics got added)
`main-plan.md` lists the original 17 topics in a research-convenient order. This syllabus regroups them by dependency: you can't understand *why* systemd/boot work the way they do without processes first; you can't understand disk/inode exhaustion without inodes first; TLS and SSH both build on the raw TCP handshake; and PAM/sudo/su/SELinux all build on the setuid and rwx mechanisms you meet in Module 1.

The original 17 topics covered "bare metal Linux" thoroughly but stopped short of two things that matter a lot for a DevOps/SRE track specifically: **containers** (namespaces/cgroups - arguably the single highest-leverage gap, since Docker/Kubernetes are built directly on top of Module 1's process model) and **DNS** (notable because it's literally the NetworkChuck video used as this syllabus's own quality benchmark, yet wasn't in the original list). The other 7 additions - OOM killer, LVM/RAID, package management, iptables/nftables, SELinux/AppArmor - round out gaps in the filesystem, networking, and privilege modules that a working sysadmin hits regularly. Containers got a whole new module (5) rather than being squeezed into an existing one, because it's genuinely the capstone of Modules 1–4: a container is just processes + namespaces + cgroups, nothing more, and that only clicks once you've done the modules it depends on. Cron moved to Module 6 as the final closing module - small, applied, and now able to be directly compared against the systemd-timer alternative from Module 2.

## Supplementary anchors (use throughout, not tied to one topic)
- **Julia Evans / wizardzines.com** - by far the biggest single contributor to this syllabus: file descriptors, permissions, inodes, TCP/IP, DNS, the OOM killer, and the entire containers module all have a matching zine or comic from her. Worth buying the "Your Linux Toolbox" bundle + the Networking and Containers zines specifically.
- **Core Dumped (YouTube)** - animated OS-internals channel; confirmed strong for processes. Periodically check its "Operating Systems Theory" playlist - if it publishes dedicated episodes on signals, env vars, or sudo/su, those would beat the current flagged picks.
- **Liz Rice, "Containers From Scratch" (GOTO 2018)** - the single best mechanism-level video for Module 5; live-codes a container from raw syscalls.
- **iximiuz Labs (Ivan Velichko)** - highly visual, interactive hands-on labs specifically for cgroups and namespaces; good pairing resource throughout Module 5.
- **Brian Ward, *How Linux Works, 3rd ed.*** - the single-book safety net for every flagged topic.
