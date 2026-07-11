---
created: 2026-07-11
updated: 2026-07-11
stage: wiki
source: "[[isobar-v2-migration-and-reliability-overhaul]]"
---

# A clean exit looks identical to success — to your supervisor

Two isobar bugs, same root cause: a process supervisor (launchd, systemd, pm2, etc.) can only see the exit code, not your intent.

- `sys.exit()` called inside a scheduler's async job doesn't kill the process — the scheduler catches it as a job exception and keeps running. A watchdog built on it "fired" but never actually terminated anything until switched to `os._exit()`.
- A daily cleanup script called `pkill` on the bot process, which produces a clean exit (code 0). The supervisor was configured to restart only on *failed* exits — so a deliberate, harmless-looking cleanup step caused multi-hour silent outages, because the supervisor correctly saw "this exited fine" and did nothing.

## The rule
- If a process should never intentionally exit except through one specific controlled path, make every other exit path return non-zero — otherwise your supervisor won't restart it when it should.
- Killing a process from outside (`pkill`, `kill`) produces a "successful" exit by default. If your supervisor treats successful exits as "don't restart," an external kill will look like an intentional shutdown.
- Test supervisor behavior by actually killing the process and timing the restart — don't assume the config does what it says.
