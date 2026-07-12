# setuid, setgid, sticky bit

Special permission bits that sit on top of the normal `rwx` for user/group/other.

## Where they live

A full `chmod` mode is 4 digits, not 3: `chmod 4755 file`. The leading digit is the special-bits mask:

| Value | Bit     |
|-------|---------|
| 4     | setuid  |
| 2     | setgid  |
| 1     | sticky  |

They can combine — e.g. `6` = setuid + setgid, `1` and `2` together = `3`, etc.

```
chmod  4    7    5    5
       │    │    │    │
       │    │    │    └── other  (rwx)
       │    │    └─────── group  (rwx)
       │    └──────────── owner  (rwx)
       └───────────────── special (4=setuid, 2=setgid, 1=sticky)
```

In `ls -l` output, they show up by replacing the `x` in a specific slot:

- setuid → owner's execute slot: `rws------` (or `rwS` if the bit is set but the underlying execute bit isn't — capital means "no execute, but the special bit is on")
- setgid → group's execute slot: `------s---` position → `rwxrws---`
- sticky → other's execute slot: `rwxrwxrwt`

---

## setuid — run as the file's owner

**Applies to:** executables only.

Normally a process runs with the privileges of the user who launched it. setuid overrides that: the process runs with the privileges of the **file's owner**, regardless of who invoked it.

**Classic example:** `/usr/bin/passwd`

```
-rwsr-xr-x  1 root root  /usr/bin/passwd
    ^
    "s" here instead of "x"  →  setuid is on
```

```
   you (uid 1000)
        │
        │ runs passwd
        ▼
   ┌────────────────────┐
   │       passwd        │   setuid bit set, file owned by root
   │  (executes AS root,  │   ────────────────────────────────►
   │   not as you)        │
   └────────────────────┘
        │
        │ writes new password hash
        ▼
   /etc/shadow   (normally root:root, mode 0600)
```

A regular user needs to update `/etc/shadow`, which only root can write. `passwd` is owned by root and has setuid set, so when *any* user runs it, it executes as root just long enough to write the new password hash, then exits.

Without setuid here, you'd need either:
- `/etc/shadow` world-writable (bad — anyone could edit anyone's password hash), or
- `sudo passwd` for every password change (workable, but pushes a routine self-service action through the sudo/audit path unnecessarily)

**Security note:** any setuid-root binary is a privilege escalation target. A shell-out, path injection, or buffer overflow bug in that binary can hand an attacker root. Standard hardening practice is to enumerate them:

```bash
find / -perm -4000 -type f 2>/dev/null
```
and confirm every result is expected. Worth keeping as a piece of audit evidence for hardening/least-privilege controls.

---

## setgid — run as the file's group, or inherit the directory's group

**Applies to:** executables, directories. No effect on plain files on modern Linux.

### On an executable
Same idea as setuid, one level down — the process runs with the **file's group**, not the invoking user's group. Less commonly used than setuid.

### On a directory (the one you'll actually reach for)
Normally, when a user creates a file, it gets the user's *primary group*. With setgid set on the parent directory, every new file or subdirectory created inside it inherits the **directory's group** instead — no matter who creates it.

```
drwxrwsr-x  root devteam  /var/www/shared_project
        ^
        "s" here instead of "x"  →  setgid is on
```

```
   alice (primary group: alice)  ──┐
                                     │ creates a file in shared_project/
   bob   (primary group: bob)    ──┤
                                     ▼
                          ┌───────────────────────────┐
                          │   shared_project/           │  setgid set, group = devteam
                          │   every new file inherits →  │
                          │   group "devteam"             │  (not alice's or bob's own
                          └───────────────────────────┘   primary group)
```

```bash
chmod 2775 /var/www/shared_project
chown :devteam /var/www/shared_project
```

Now `alice` (primary group `alice`) and `bob` (primary group `bob`) can both create files in `shared_project/`, and every file lands in group `devteam` automatically — no chasing down files created with the wrong group, no manual `chgrp` after the fact. This is the standard pattern for any shared team/deploy directory.

---

## Sticky bit — restrict deletion, not creation

**Applies to:** directories (files, historically used for something else, ignored today).

Normally, if you have write permission on a directory, you can delete or rename *any* file in it — permission to delete a file is a property of the **directory**, not the file itself. The sticky bit narrows that: in a sticky directory, a user can only delete or rename files **they own**, even though the directory is otherwise world-writable.

```
drwxrwxrwt  root root  /tmp
          ^
          "t" here instead of "x"  →  sticky bit is on
```

```
   /tmp   (world-writable, sticky bit set)
     │
     ├── alice/file.tmp   ── alice can delete this: OK (she owns it)
     │
     └── bob/file.tmp     ── alice CANNOT delete this: denied
                              (dir is world-writable, but sticky bit
                               restricts delete to the file's own owner)
```

```bash
chmod 1777 /tmp
```

`/tmp` has to be writable by everyone (any user, any process needs to drop temp files there), but you don't want user `bob` able to delete user `alice`'s temp files just because the directory is world-writable. Sticky bit fixes exactly that gap.

---

## Quick reference

| Bit     | Numeric | Applies to        | Effect                                                    | Symbol in `ls -l`      |
|---------|---------|--------------------|-------------------------------------------------------------|--------------------------|
| setuid  | 4       | executables        | runs as file **owner**                                     | `s` in owner's x slot   |
| setgid  | 2       | executables, dirs  | executable: runs as file **group** · dir: new files inherit dir's group | `s` in group's x slot |
| sticky  | 1       | directories        | only the file's **owner** can delete/rename it              | `t` in other's x slot  |

## Practical relevance for infra work

- Audit `find / -perm -4000` and `find / -perm -2000` periodically — unexpected setuid/setgid binaries are a red flag. **==> This is important**
- Use setgid on shared deploy/project directories (e.g. Ansible-managed paths multiple team members write to) instead of manually fixing group ownership after every write.
- `/tmp` and `/var/tmp` should always carry the sticky bit — verify this as part of any hardening baseline.