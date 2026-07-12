« [Back to syllabus](syllabus.md) · Prerequisite: [Module 1](module-1-process-fundamentals.md)

# Module 2 - Filesystem & Boot

Inodes underpin the disk-exhaustion failure mode; understanding processes (Module 1) is what makes systemd and the boot sequence actually make sense rather than being a list of stage names.

---

## 7. Inodes, hard links vs symlinks
- [ ] Done

**Resource:** Julia Evans "Bite Size Linux" zine (illustrated) - https://wizardzines.com/zines/bite-size-linux/ - paired with "What is: hard link, symlink, and inode in Linux?" (RTFM) - https://rtfm.co.ua/en/what-is-hard-link-symlink-and-inode-in-linux/

**What you'll learn:** A hard link is a second directory entry pointing at the *same inode number* (link count goes up); a symlink is a *separate* inode that just stores a path string. This is why hard links can't cross filesystems (inode numbers are only unique within one FS) - and it's the direct cause of the "df shows free space but I still get 'No space left on device'" failure in topic 10.

**Confirm it:**
```
touch a && ln a b && ln -s a c
ls -li a b c
```
Confirm `a` and `b` share the same inode number and link count 2; `c` has its own (different) inode and shows as a symlink (`l` in the mode bits).

---

## 8. LVM and RAID - volumes underneath the filesystem
- [ ] Done

**⚠️ Partially flagged - no comic-style resource found; closest is a whiteboard-diagram video, not a drawn explainer.**

**Resource:** "LVM on Linux: The Ultimate Beginner's Guide" (whiteboard explanation + hands-on demo) - https://www.youtube.com/watch?v=I2nLXQ16XsA - paired with "Beginners guide to how LVM works in Linux (architecture)" - GoLinuxCloud - https://www.golinuxcloud.com/overview-lvm-in-linux/ for the PV → VG → LV layering diagrams.

**What you'll learn:** LVM adds an abstraction layer *below* the filesystem: physical volumes (PVs, real disks/partitions) get pooled into a volume group (VG), which is then sliced into logical volumes (LVs) that behave like partitions but can be resized, snapshotted, or striped/mirrored across disks without touching the filesystem sitting on top. RAID (0/1/5/6/10) either lives underneath LVM (mdadm) or inside it (LVM-RAID) for redundancy or performance.

**Confirm it:**
```
sudo pvcreate /dev/loop0 /dev/loop1   # or real spare disks
sudo vgcreate testvg /dev/loop0 /dev/loop1
sudo lvcreate -L 100M -n testlv testvg
sudo lvs && sudo vgs && sudo pvs
```
(Use `losetup` to create loopback devices from files if you don't have spare disks: `dd if=/dev/zero of=disk1.img bs=1M count=200 && sudo losetup /dev/loop0 disk1.img`.) Confirm you can see the PV→VG→LV chain, then `sudo lvextend -L +50M testvg/testlv` and observe the LV grow without touching any filesystem.

---

## 9. Filesystem Hierarchy Standard (why /etc, /var, /usr exist as they do)
- [ ] Done

**Resource:** "Understanding the bin, sbin, usr/bin, usr/sbin split" - Rob Landley - https://lists.busybox.net/pipermail/busybox/2010-December/074114.html

**What you'll learn:** The `/usr` split is a 1970s accident - the OS outgrew its first disk pack and "leaked" into the second one, which happened to hold user home directories (hence "usr"). Landley's blunt conclusion: it "stopped making any sense before Linux was ever invented" but got carried forward by convention.

**Confirm it:**
```
ls -l /bin
```
On most modern distros this prints something like `/bin -> usr/bin` - confirm your system has already merged the split Landley describes as historically pointless.

---

## 10. Disk and inode exhaustion - real failure mechanics
- [ ] Done

**⚠️ Flagged - no single analogy+mechanism+visual resource exists.**

**Resource (pair):** `core(5)` man page - https://man7.org/linux/man-pages/man5/core.5.html - plus the RTFM article from topic 7 for the inode-exhaustion mechanism itself.

**What you'll learn:** Inodes are a *fixed count* set at filesystem creation time. Run out of them and you get "No space left on device" even while `df -h` shows free blocks - because you've run out of the metadata slots needed to describe new files, not out of data space.

**Confirm it:**
```
df -h /
df -i /
```
Compare block usage (`-h`) vs inode usage (`-i`) on the same filesystem - they're independent counters and can diverge significantly, e.g. many small files eating inodes while blocks stay free.

---

## 11. Package management internals (apt/dpkg)
- [ ] Done

**⚠️ Flagged - no illustrated resource found; mostly cheatsheet-style tutorials.**

**Resource:** Debian Reference, Chapter 2 - "Debian package management" - https://www.debian.org/doc/manuals/debian-reference/ch02.en.html

**What you'll learn:** `dpkg` is the low-level tool that actually unpacks a `.deb`, runs its maintainer scripts, and registers it in `/var/lib/dpkg/status` - but it doesn't resolve dependencies. `apt` sits above it: it resolves the dependency graph, does a topological sort so packages install in the right order, downloads from repositories defined in your sources list, then hands each `.deb` to `dpkg` to actually apply. Every installed file you can find later with `dpkg -L <package>` traces back to this two-layer split.

**Confirm it:**
```
dpkg -L bash | head -5
dpkg -S /bin/ls
apt-cache depends bash | head -5
```
Confirm you can go both directions: from a package name to the files it owns, and from a file back to the package that owns it. Then look at `apt-cache depends` to see the dependency graph `apt` had to resolve before it ever called `dpkg`.

---

## 12. systemd units, targets, and dependency ordering
- [ ] Done

**Resource:** "Rethinking PID 1" - Lennart Poettering - https://0pointer.de/blog/projects/systemd.html

**What you'll learn:** Written by systemd's creator - units are typed, file-backed objects (service, mount, device, target, automount…), and dependency ordering is often *implicit*: a mount of `/home/lennart` implicitly depends on the mount of `/home`, with the kernel itself synchronizing device/mount relationships without userspace management.

**Confirm it:**
```
systemctl list-dependencies sshd
```
(Substitute any running service if `sshd` isn't installed.) Identify at least one dependency that's a `.mount` or `.socket` unit rather than another `.service` - a sign of the typed-unit model in action.

---

## 13. The Linux boot process end to end
- [ ] Done

**Resource:** "An introduction to the Linux boot and startup processes" - David Both, Opensource.com - https://opensource.com/article/17/2/linux-boot-and-startup - with Brian Ward's *How Linux Works, 3rd ed.* as the deeper reference.

**What you'll learn:** Firmware/POST → GRUB2 → kernel decompression + initramfs → systemd → targets. The reasoning for initramfs specifically: mounting the *real* root filesystem often needs drivers (LVM, RAID, encryption) that themselves live on that root filesystem - a temporary RAM filesystem breaks the chicken-and-egg problem.

**Confirm it:**
```
systemd-analyze blame
journalctl -b | head -30
```
Identify the slowest-starting unit from `blame`, then find its startup line in the boot journal and note roughly where in the sequence (early boot vs late target) it falls.

---

## 🧪 Module 2 capstone - lock it in on a throwaway VM

On a fresh disposable VM with at least two extra virtual/loopback disks attached:

1. Build an LVM volume group across both disks, create a logical volume, and format it `ext4`. Mount it.
2. Deliberately exhaust its **inodes**, not its space: `for i in $(seq 1 100000); do touch /mnt/testlv/f$i; done` on a small LV until you hit "No space left on device" - then run `df -h` and `df -i` side by side and confirm blocks are still free while inodes are gone.
3. `apt install` a small package you don't have, then use `dpkg -L` to list every file it dropped and `systemctl status` to check whether it registered a new unit.
4. Reboot the VM and run `systemd-analyze blame` + `journalctl -b` again - find where in the boot sequence your new LV gets mounted (look for its `.mount` unit) relative to other units.
5. Delete the VM.
