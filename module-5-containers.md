« [Back to syllabus](syllabus.md) · Prerequisite: [Module 1](module-1-process-fundamentals.md) (processes, permissions), [Module 3](module-3-networking.md) (network namespaces)

# Module 5 — Containers & Modern Infra

This is the payoff module for the DevOps/SRE track: a container is not a new kernel feature — it's processes (Module 1) wrapped in namespaces (isolation) and cgroups (resource limits), which is exactly why this module comes last among the "core Linux" modules rather than first. If you understand Modules 1–4, this module should feel like watching familiar pieces get assembled rather than learning something brand new.

---

## 23. Linux namespaces — the isolation primitive
- [ ] Done

**Resource:** "How Containers Work!" — Julia Evans (zine) — https://wizardzines.com/zines/containers/ — comics: "namespaces" — https://wizardzines.com/comics/namespaces/ and "user namespaces" — https://wizardzines.com/comics/user-namespaces/ — paired with "Containers From Scratch" — Liz Rice, GOTO 2018 — https://www.youtube.com/watch?v=8fi7uSYlOdc (live-codes a container in ~40 lines of Go).

**What you'll learn:** A namespace doesn't add anything — it *hides* things. A PID namespace makes a process see itself as PID 1 with no visibility into processes outside it; a mount namespace gives it its own view of the filesystem tree; a network namespace gives it its own network stack (which you already touched in Module 3, topic 16). Liz Rice's talk shows the actual `unshare()`/`clone()` syscalls doing this live, one namespace at a time, so you watch a normal process become "container-like" incrementally.

**Confirm it:**
```
unshare --pid --fork --mount-proc bash
ps aux
```
Inside the new shell, confirm `ps aux` shows almost nothing — your shell thinks it's PID 1 in its own tiny process universe. Exit and run plain `ps aux` again to see your real process list return.

---

## 24. cgroups — the resource-limiting primitive
- [ ] Done

**Resource:** "containers aren't magic" — Julia Evans (comic) — https://wizardzines.com/comics/containers-arent-magic/ (shows building a mini-container in 15 lines of bash, cgroups included) — paired with "Get Started with Linux Control Groups (cgroup v2)" — iximiuz Labs — https://labs.iximiuz.com/skill-paths/get-started-with-cgroups (interactive, hands-on).

**What you'll learn:** Where namespaces control *what a process can see*, cgroups control *what it can use* — CPU shares, memory ceilings, I/O bandwidth, max process count — enforced through a virtual filesystem (`/sys/fs/cgroup/`), not a syscall API. This is the mechanism behind `docker run --memory=512m`, and it's also what actually enforces the OOM behavior you triggered manually in Module 1's capstone — cgroup memory limits trigger their own scoped OOM kills independent of the whole-system one.

**Confirm it:**
```
cat /sys/fs/cgroup/cgroup.controllers
sudo mkdir /sys/fs/cgroup/test
echo "10000" | sudo tee /sys/fs/cgroup/test/memory.max
```
Confirm you can see the cgroup v2 controllers available on your system, then create a cgroup and set a (deliberately tiny, 10KB) memory ceiling on it — you'll actually use it to constrain a process in this module's capstone below.

---

## 25. Container runtime internals — how it all becomes `docker run`
- [ ] Done

**Resource:** "containers = processes" — Julia Evans (comic) — https://wizardzines.com/comics/containers-are-processes/ — same "Containers From Scratch" talk from topic 23 for the mechanism (Liz Rice adds `chroot`/pivot_root on top of namespaces + cgroups in the second half of the talk).

**What you'll learn:** A container image is a tarball of filesystem layers (usually merged with OverlayFS at runtime — the same union-mount idea, one read-only layer per image layer plus one writable layer on top); a container runtime (runc, under Docker/containerd) combines namespaces + cgroups + a `pivot_root`/chroot into that merged filesystem + dropped capabilities into one OCI-spec'd process launch. `docker run` is a thin client that ultimately just... does what you did manually in topics 18–19, plus stitches in the filesystem layer.

**Confirm it:**
```
docker run --rm alpine cat /proc/1/status | grep -i pid
docker inspect --format '{{.State.Pid}}' <a-running-container>
```
Confirm the container thinks its main process is PID 1 (from `/proc/1/status` inside it), while `docker inspect` on the host shows a *real*, different, much larger PID — the same process, seen through two different PID namespaces.

---

## 🧪 Module 5 capstone — build a "container" by hand on a throwaway VM

On a disposable VM (needs to be a real or nested-virtualization VM — containers use kernel features directly, no special hardware, but you do need root and a recent kernel):

1. Download a minimal root filesystem (e.g. `alpine-minirootfs`) and extract it to `/tmp/myrootfs`.
2. Use `unshare --pid --mount --uts --net --fork` combined with `chroot /tmp/myrootfs /bin/sh` to land a shell that has its own PID namespace, its own mount namespace, its own hostname (uts), its own (empty) network stack, and its own root filesystem — by hand, no Docker involved.
3. From a second terminal on the host, create a cgroup and set a memory limit on it (as in topic 24), then move your chrooted shell's PID into that cgroup (`echo <pid> | sudo tee /sys/fs/cgroup/test/cgroup.procs`) and confirm a memory-hungry command inside the chroot gets killed once it crosses the limit.
4. Now install Docker on the same VM, run `docker run --memory=10m alpine sh -c 'stress-ng --vm 1 --vm-bytes 50m'` (or similar), and use `docker inspect` + `cat /sys/fs/cgroup/**/memory.max` to find the *exact same primitives* you just built by hand, now driven by the Docker daemon instead of your own `unshare`/cgroup commands.
5. Destroy the VM — you've now seen "what is a container" from first principles instead of taking it on faith.
