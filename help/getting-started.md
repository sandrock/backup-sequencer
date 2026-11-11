Getting started
================================

End-user guide to installing, configuring, scheduling, and running the sequencer.


Requirements
----------------------------------------------------------------

- PowerShell 7 (`pwsh`).
- `Pester` (for running the tests).
- Per-service tools your steps need (e.g. a container runtime, `tar`).
- `swaks` on the host if you want email reports.


Where things go
----------------------------------------------------------------

| | Linux | Windows |
|---|---|---|
| Code (this repo) | `/opt/backup-sequencer` | `C:\Program Files\backup-sequencer` |
| Config | `/etc/backup-sequencer/` | `%ProgramData%\backup-sequencer\` |
| State (events, lock, staging) | `/var/lib/backup-sequencer/` | `%ProgramData%\backup-sequencer\var\` |

All three are overridable with `BACKUP_SEQUENCER_CONFIG_DIR` and
`BACKUP_SEQUENCER_VAR_DIR`. Deploy the code read-only (git pull / rsync /
package); keep config and secrets out of the code tree.


Where backups land
----------------------------------------------------------------

Each service's dump is produced under a *staging* directory:

```
<staging-root>/<service>/wip/<archive>
```

The staging root is resolved as:

1. `staging` in `config.json`, if set; otherwise
2. `<state dir>/staging` — i.e. `/var/lib/backup-sequencer/staging` (Linux) or
   `%ProgramData%\backup-sequencer\var\staging` (Windows).

Set `staging` explicitly to a path with room for a service's largest dump —
especially if the state dir is on a small system disk. Put it on the volume that
has the space. A shipping step, if present, reads from here; otherwise the recipe's
`cleanup` frees it after the run.


Configure
----------------------------------------------------------------

Copy `config.sample.json` to the config dir as `config.json` and edit it:

- `window` — the time window (`HH:mm`), may wrap past midnight.
- `services` — one entry per service. Set `mandatory: true` for the ones that
  must always run first. Point `script` at a recipe or a bespoke service script,
  and pass that script's parameters in `env`.
- `notify` — set `enabled: true` and your SMTP details; put the password in the
  file named by `passwordFile` (relative to the config dir), mode `0600`.

Each step may set `retries`, `timeoutSec`, and `alwaysRun` (run even after an
earlier step in the same service failed — use it for recovery/cleanup).


Run
----------------------------------------------------------------

```sh
pwsh <install>/src/sequencer.ps1 plan       # show the plan, no side effects
pwsh <install>/src/sequencer.ps1 run        # execute (respects the window)
pwsh <install>/src/sequencer.ps1 run -Force # execute now, ignore the window
pwsh <install>/src/sequencer.ps1 history    # recent runs, newest first
```


Schedule
----------------------------------------------------------------

Run once at window open; the sequencer stops starting new work at window close.

- Linux (systemd): a `.timer` with `OnCalendar` at the window open, triggering a
  oneshot service that runs `sequencer.ps1 run`. The single-instance lock makes a
  double-trigger a no-op.
- Windows (Task Scheduler): a daily trigger at the window open running
  `pwsh.exe <install>\src\sequencer.ps1 run`.


Reports
----------------------------------------------------------------

With `notify.enabled`, each run emails a terse digest (services, status,
durations, sizes) via `swaks`. Sending every run acts as a dead-man's-switch: no
email means the run did not happen. Keep the body free of secrets.
