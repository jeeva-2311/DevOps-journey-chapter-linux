« [Back to syllabus](syllabus.md)

# Module 1 — Process & OS Fundamentals

Everything downstream (systemd, boot, networking, auth, cron) is built on top of processes, file descriptors, permissions, signals, and environment variables. Start here.

---

## 1. Processes, fork/exec, zombie/orphan processes
- [ ] Done

**Resource:** "The Weird Way Linux Creates Processes" — Core Dumped (animated) — https://www.youtube.com/watch?v=SwIPOf2YAgI
For the zombie/orphan lifecycle specifically, pair it with the Wikipedia "Zombie process" article (accurate, not illustrated).

**What you'll learn:** Why Unix creates processes by *cloning then replacing* (fork then exec) rather than spawning directly, and what actually happens to a process's process-table entry between its own death and its parent "reaping" it.

**Confirm it:**
```
sleep 100 &
ps -o pid,ppid,stat,cmd
kill %1
```
Identify which PID is the parent shell and which is the child `sleep`. Then run a `sleep 1 &` and immediately `ps -o pid,ppid,stat` again fast enough to try to catch its `Z` (zombie) state before the shell reaps it.

---

## 2. File descriptors and "everything is a file"
- [ ] Done

**Resource:** "File descriptors" — Julia Evans (comic) — https://wizardzines.com/comics/file-descriptors/

**What you'll learn:** The kernel hands your process integer "tickets" — fd 0/1/2 are stdin/stdout/stderr, and a file descriptor can point at a real file, a pipe, or a socket; "read from stdin" just means "read from fd 0."

**Confirm it:**
```
ls -l /proc/$$/fd
```
Identify which numbers are your terminal's stdin/stdout/stderr. Then run `ls -l /proc/$$/fd < /etc/hostname` in a subshell and notice fd 0 now points somewhere different.

---

## 3. Linux permissions (rwx, UID/GID, setuid/setgid, sticky bit)
- [ ] Done

**Resource:** "Permissions" — Julia Evans (comic) — https://wizardzines.com/comics/permissions/

**What you'll learn:** Permissions are really 12 bits, not 9 — setuid/setgid/sticky are the top three. `ls -l /usr/bin/ping` shows `rws r-x r-x root root` — the `s` means ping always runs as root regardless of who launches it. This is the exact mechanism `sudo`/`su` rely on (see Module 4).

**Confirm it:**
```
stat -c '%a %U' /usr/bin/ping
```
Confirm the leading digit is 4xxx (setuid bit set) and the owner is root. Compare against `stat -c '%a %U' /bin/ls` (no setuid bit).

---

## 4. Signals (SIGTERM, SIGKILL, SIGHUP, etc.)
- [ ] Done

**⚠️ Flagged — no single resource fully meets the bar.** No confirmed illustrated/animated explainer covering signal delivery, pending/blocked masks, and why SIGKILL/SIGSTOP are uncatchable.

**Resource:** GNU C Library manual, "Termination Signals" — https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html (text, but the most accurate primary source on *why* each signal behaves as it does).

**What you'll learn:** Why SIGQUIT deliberately preserves temp files for post-mortem debugging, and why SIGKILL "cannot be handled or ignored, and is therefore always fatal" — it never reaches userspace at all.

**Confirm it:**
```
bash -c 'trap "echo caught TERM" TERM; kill -TERM $$; sleep 1'
bash -c 'trap "echo caught KILL" KILL; kill -KILL $$; echo "did this print?"'
```
Confirm the first prints "caught TERM" but the second process just dies silently — SIGKILL never gives the handler a chance to run.

---

## 5. The OOM killer, memory pressure, and swap
- [ ] Done

**Resource:** "the OOM killer" — Julia Evans (comic) — https://wizardzines.com/comics/oom-killer/ — paired with "Linux OOM Killer: A Detailed Guide to Memory Management" — Last9 — https://last9.io/blog/understanding-the-linux-oom-killer/ for the scoring mechanism in prose.

**What you'll learn:** When the system runs critically low on memory, the kernel's `select_bad_process()` picks a victim using a "badness" score (`rss_pages + swap_pages + page_table_pages`, weighted by `oom_score_adj`) — not randomly. It prefers killing children over parents (the "sacrifice child" principle), and you can protect or condemn a specific process via `/proc/<pid>/oom_score_adj`. This directly builds on topic 4: SIGKILL is exactly what the OOM killer sends.

**Confirm it:**
```
cat /proc/self/oom_score_adj
echo -1000 | sudo tee /proc/$$/oom_score_adj
cat /proc/self/oom_score_adj
```
Confirm you can read and change your own shell's OOM protection score. (Don't actually trigger an OOM here — you'll do that deliberately, safely, on a throwaway VM in this module's capstone below.)

---

## 6. Environment variables and shell inheritance
- [ ] Done

**⚠️ Flagged — no single resource fully meets the bar.** Mostly shallow "how to set an env var" tutorials; nothing walks the `environ` array / `execve()` mechanism visually.

**Resource:** Julia Evans' "So You Want To Be A Wizard" / "Bite Size Command Line" zines (partial coverage) + Brian Ward's *How Linux Works, 3rd ed.* (accurate prose treatment of the inheritance mechanism).

**What you'll learn:** A child process gets a *copy* of its parent's environment at `execve()` time, which is why `export X=1` inside a subshell can never affect the parent shell.

**Confirm it:**
```
export TESTVAR=parent
bash -c 'export TESTVAR=child; echo "child sees: $TESTVAR"'
echo "parent sees: $TESTVAR"
```
Confirm the parent still says "parent" — the child's `export` only ever mutated its own copy.

---

## 🧪 Module 1 capstone — lock it in on a throwaway VM

Spin up a disposable VM (a cheap DigitalOcean/Hetzner box, or a local VM via multipass/VirtualBox/Vagrant — anything you can destroy afterward). In one session:

1. Write a tiny script that forks a background child, `trap`s `SIGTERM` in the parent to log a message and clean up, then sends itself `SIGTERM` — confirm the trap fires. Then try the same with `SIGKILL` and confirm nothing can catch it.
2. Install `stress-ng` and deliberately exhaust the VM's memory (`stress-ng --vm 2 --vm-bytes 95% --timeout 30s`). Watch `dmesg -T | grep -i "out of memory"` (or `journalctl -k`) in a second terminal and identify which process the kernel actually killed and why (check its `oom_score` beforehand with `cat /proc/<pid>/oom_score`).
3. Before re-running the stress test, set `oom_score_adj` to `-1000` on a process you want to protect (e.g. your SSH session's shell) and confirm it survives the next OOM event while unprotected processes don't.
4. Throughout, use `ls -l /proc/<pid>/fd` and `stat` on a setuid binary to sanity-check topics 2–3 are still making sense in a live environment, not just your own laptop.

Destroy the VM when done — the point was the observation, not the box.
