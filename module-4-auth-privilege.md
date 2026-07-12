« [Back to syllabus](syllabus.md) · Prerequisite: [Module 1](module-1-process-fundamentals.md) (permissions/setuid)

# Module 4 — Authentication & Privilege

PAM is the general pluggable auth framework; sudo/su is a concrete consumer of it, built on the setuid mechanism you met in Module 1. SELinux/AppArmor close the module out as the mandatory-access-control layer sitting on top of the discretionary (rwx) model from Module 1.

---

## 20. PAM (Pluggable Authentication Modules) — what it is and why
- [ ] Done

**Resource:** "An introduction to Pluggable Authentication Modules (PAM) in Linux" — Susan Lauber, Red Hat blog — https://www.redhat.com/en/blog/pluggable-authentication-modules-pam — companion: "Anatomy of a PAM configuration file" — https://www.redhat.com/en/blog/pam-configuration-file

**What you'll learn:** Historically every program rolled its own authentication logic; PAM (from Solaris, ~1997) centralizes auth behind one documented library, letting admins swap or stack auth sources (Kerberos, SSSD, local files) per-service via `/etc/pam.d`, without any application code changes. The account/auth/password/session module types and stacking/control-flag model.

**Confirm it:**
```
cat /etc/pam.d/sudo
```
Identify at least one line's module type (`auth`, `account`, `password`, `session`) and control flag (`required`, `sufficient`, etc.), and name the module it's stacking in.

---

## 21. sudo vs su and the Linux privilege model
- [ ] Done

**⚠️ Flagged — no single resource meets the bar.** Most content is command cheatsheets, not the underlying privilege model.

**Resource (pair):** the setuid explanation in Julia Evans' permissions comic (Module 1, topic 3) — shows the exact mechanism `sudo`/`su` both rely on — plus the `sudo` and `sudoers` man pages for policy/logging specifics. Brian Ward's *How Linux Works* covers the full privilege model (real vs effective UID) in one place.

**What you'll learn:** `su` spawns a full shell as another user; `sudo` instead consults a policy file (`/etc/sudoers`), runs just the one command with elevated privilege, and logs the invocation — both are ultimately built on setuid-root binaries and the real-vs-effective-UID distinction.

**Confirm it:**
```
id
sudo -l
su - -c 'id'
```
Compare the `uid`/`gid` reported by plain `id` against what `su -c 'id'` reports, and read what `sudo -l` says you're permitted to run without actually running anything privileged.

---

## 22. SELinux vs AppArmor — mandatory access control
- [ ] Done

**⚠️ Flagged — no illustrated resource found; comparison articles only.**

**Resource:** "Technologies for container isolation: A comparison of AppArmor and SELinux" — Red Hat blog — https://www.redhat.com/en/blog/apparmor-selinux-isolation

**What you'll learn:** Everything so far in this module is *discretionary* access control (DAC) — the file owner decides who gets access. MAC is a second, mandatory layer the kernel enforces regardless of what the owner allows: AppArmor is path-based (a profile says "this binary may only read/write these paths"), while SELinux labels every file, process, and port with a security context and decides access by the relationship between labels, independent of the filesystem path. This is *why* a correctly-`chmod`ed file can still get "Permission denied" — MAC is a second gate on top of the DAC model from Module 1.

**Confirm it:**
```
getenforce 2>/dev/null || aa-status 2>/dev/null
```
Run whichever applies to your distro (SELinux's `getenforce` on RHEL/Fedora, AppArmor's `aa-status` on Ubuntu/Debian) and confirm which mode you're in (enforcing/permissive, or complain/enforce) and how many profiles/policies are currently loaded.

---

## 🧪 Module 4 capstone — lock it in on a throwaway VM

On a disposable VM:

1. Create a low-privilege user, then write a `/etc/sudoers.d/` rule (via `visudo -f`) that lets that user run exactly *one* command as root — nothing else. Confirm with `sudo -l -U <user>` and by testing that a different command correctly gets refused.
2. Read `/etc/pam.d/sudo` on this VM and identify the module stack enforcing that policy.
3. If your distro ships AppArmor: put a small custom script in complain mode (`aa-complain`), run it doing something borderline (e.g. writing outside its expected directory), and check `journalctl` / `dmesg` for the audit log entry AppArmor would have blocked in enforce mode. (RHEL/Fedora users: do the SELinux equivalent with `semanage` + `ausearch`.)
4. Destroy the VM.
