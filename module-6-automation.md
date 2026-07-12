« [Back to syllabus](syllabus.md) · Prerequisite: Modules 1–5

# Module 6 - Automation

The closing module: cron pulls together processes, permissions, and locking from everything before it, and now that you know systemd (Module 2) and containers (Module 5), you can compare cron's failure modes against the alternatives directly.

---

## 26. Cron internals and why race conditions happen
- [ ] Done

**⚠️ Partially flagged.** Mechanism-first and accurate, but it's a blog post covering the locking/race angle more than cron's internal scheduling implementation - the best available rather than a flawless match.

**Resource:** "Prevent cronjobs from overlapping in Linux" - Mattias Geniar - https://ma.ttias.be/prevent-cronjobs-from-overlapping-in-linux/

**What you'll learn:** A job scheduled every minute that takes longer than a minute starts overlapping itself - if it touches shared data, you get corruption plus a load-spiral. The real fix is `flock` advisory locks: "the kernel releases the lock automatically when the process exits," unlike a hand-rolled PID lockfile that goes stale after `kill -9` or a crash. (Bonus: systemd timers avoid this class of bug entirely - they won't start a second instance while one is still running.)

**Confirm it:**
```
* * * * * /usr/bin/flock -n /tmp/mycron.lock /path/to/script.sh
```
Write this as a crontab entry (`crontab -e`) pointing at any script that sleeps for 90 seconds. Watch two consecutive runs - the second one should skip (because `-n` fails fast if the lock is held) instead of stacking on top of the first. Explain to yourself, out loud, why removing `flock -n` from that line would let both instances run concurrently.

---

## 🧪 Module 6 capstone - lock it in on a throwaway VM

On a disposable VM:

1. Write a naive cron job (no locking) that runs every minute but sleeps for 90 seconds. Let it run for 3–4 minutes and use `pgrep -a` (or `ps aux`) to catch two instances genuinely overlapping - take a screenshot or paste the output so you have proof, not just the theory.
2. Fix it with the `flock -n` pattern from topic 26 and confirm overlapping stops - the second invocation now exits immediately instead of stacking.
3. Recreate the exact same job as a systemd service + timer unit pair instead (`.service` + `.timer`, using what you learned in Module 2) and confirm via `systemctl list-timers` and `journalctl -u <yourjob>` that systemd refuses to start a second instance on its own, with no `flock` needed.
4. Write one sentence for yourself on when you'd still reach for cron over a systemd timer in 2026 (portability to non-systemd systems, simplicity for a one-off, etc.) - this is the actual judgment call you'll be making on the job.
5. Destroy the VM. This closes out the syllabus - from here, the fastest way to keep the knowledge is to hit these mechanisms again for real, under a bit of production pressure.
