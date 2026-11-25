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
| State (events, lock) | `/var/lib/backup-sequencer/` | `%ProgramData%\backup-sequencer\var\` |
| Spool (transient backups) | `/var/spool/backup-sequencer/` | `%ProgramData%\backup-sequencer\spool\` |

All are overridable with `BACKUP_SEQUENCER_CONFIG_DIR`, `BACKUP_SEQUENCER_VAR_DIR`,
and `BACKUP_SEQUENCER_SPOOL_DIR`. Keep config and secrets out of the code tree.


Install
----------------------------------------------------------------

`packages/install.sh` installs the tool system-wide on Linux. It needs `pwsh`
already present (it does not install PowerShell). It puts the code in
`/opt/backup-sequencer`, a `bseq` command in `/usr/local/bin`, creates the
config/state/spool dirs above, and — when systemd is present — installs the
service/timer units (disabled).

Download and install the latest `main`:

```sh
curl -fsSL https://raw.githubusercontent.com/sandrock/backup-sequencer/main/packages/install.sh | sudo bash
```

Or install from a local checkout (air-gapped, or a specific branch):

```sh
sudo ./packages/install.sh --from .
```

Flags: `--no-systemd` skips the service/timer units (otherwise installed, disabled,
when systemd is present — see [packages/systemd/](../packages/systemd/README.md)),
`--ref <branch|tag>` selects what to download, `--yes` skips the prompt, `--help`
lists everything. The install locations are overridable with `BSEQ_CODE_DIR`,
`BSEQ_BIN_DIR`, `BSEQ_CONFIG_DIR`, `BSEQ_VAR_DIR`, `BSEQ_SPOOL_DIR` (handy for a dry
run into a temp prefix). Re-running the installer updates the code in place; an
existing `config.json` and existing units are never touched.

The `bseq` wrapper runs the tool under `pwsh` with quiet logging, so `bseq plan` /
`bseq run` work from anywhere once installed.


Where backups land
----------------------------------------------------------------

Each service's backup is produced and held under a *spool* root, which has phase
dirs; a backup moves through them by atomic rename:

```
<spool-root>/
  wip/<service>/      being built
  stage/<service>/    complete; an optional sync-archive step runs here
  ready/<service>/    done; waiting for the async archiver to take it
  trash/              deletions, reaped at the next run
```

The spool root is resolved as:

1. `spool.path` in `config.json`, if set; otherwise
2. `/var/spool/backup-sequencer` (Linux) or `%ProgramData%\backup-sequencer\spool`
   (Windows) — override with `BACKUP_SEQUENCER_SPOOL_DIR`.

```json
"spool": { "path": "/srv/backup-spool" }
```

Set `spool.path` explicitly to a path with room for a service's largest backup —
especially if the state dir is on a small system disk. Put it on the volume that
has the space. Before each service, the sequencer checks the spool has room for
it (see [the free-space check](configuring-services.md#spool-free-space-check)).
`spool` is an object: the older plain-string form (`"spool": "/path"`) is refused,
every command stops with the line to write instead. The async archiver takes completed backups from `ready/<service>/`
and deletes them; see [the spool and archiving model](../docs/archiving.md).


Configure
----------------------------------------------------------------

Copy `config.sample.json` to the config dir as `config.json` and edit it:

- `window` — the time window (`HH:mm`), may wrap past midnight.
- `services` — one entry per service. Set `mandatory: true` for the ones that
  must always run first. Point `script` at a recipe or a bespoke service script,
  and pass that script's parameters in `env`. For the full set of service fields
  see [Configuring a service](configuring-services.md), and for the bundled recipe
  the [docker-compose-cold recipe](recipe-docker-compose-cold.md). To ship each
  backup somewhere durable and prune it over time, see [Archiving](archiving.md).
- `notify` — set `enabled: true` and your SMTP details; put the password in the
  file named by `passwordFile` (relative to the config dir), mode `0600`. Check
  it with `sudo bseq test-notify` (see Reports below). Optional `swaks`: the
  swaks executable, when not on the PATH.

Each step may set `retries`, `timeoutSec`, and `alwaysRun` (run even after an
earlier step in the same service failed — use it for recovery/cleanup).


Run
----------------------------------------------------------------

```sh
bseq plan                      # show the plan, no side effects
bseq validate                  # check config + prerequisites, run nothing (exit 1 if invalid)
bseq run                       # execute (respects the window + fluid scheduling)
bseq run -ForceWindow          # run even outside the window
bseq run -ForceWindow -ForceAllJobs   # run everything now, ignore window, fit and coverage
bseq status                    # the run in progress (service, step, elapsed), or the last one
bseq show <run>...             # per-service detail of runs (id, or any unique ending of it)
bseq history                   # recent runs with their ids, newest first
```

The plan is ordered by a score (most overdue first); within a window the run packs
what fits and defers the rest to a later run — see
[the scheduling model](../docs/scheduling.md).

Without installing (from a checkout), use `pwsh src/sequencer.ps1 <command>` instead.

A run from a terminal is killed by Ctrl-C or a lost connection, leaving a stopped
service down: run it inside tmux or screen ([Running by hand](manual-runs.md)), or
start it through systemd ([Running under systemd](systemd.md)).

To follow a run while it works, or to dig into a past one, see
[Following runs](following-runs.md).


Schedule
----------------------------------------------------------------

Run once at window open; the sequencer stops starting new work at window close.
The run is not a daemon — it is invoked, enforces the window, and exits — so a
scheduler fires it, it does not sleep until the window.

- Linux (systemd): sample `.service` + `.timer` units are in
  [`packages/systemd/`](../packages/systemd/README.md) — copy them to
  `/etc/systemd/system/`, adjust paths and schedule, then
  `systemctl enable --now backup-sequencer.timer`. The single-instance lock makes
  a double-trigger a no-op. Day-to-day control (start a run, follow it, the
  journal, pausing the timer): [Running under systemd](systemd.md).
- Windows (Task Scheduler): a daily trigger at the window open running
  `pwsh.exe <install>\src\sequencer.ps1 run`.


Reports
----------------------------------------------------------------

With `notify.enabled`, each run emails its report via `swaks`: the same lines as
`bseq show <run>` (per service the outcome, size, archive id, step durations,
skip reasons and warnings, then the deletions). The subject carries the outcome,
`backup OK ok=5 failed=0 <run>` or `backup FAIL ...`. Sending every run acts as a
dead-man's-switch: no email means the run did not happen. The report holds no
secrets, but it names your services and paths: mind where the mail goes.

To check the settings without waiting for a run:

```sh
sudo bseq test-notify
```

It mails the last run's report with `[test]` in the subject (or a short text when
no run is recorded yet), even when `notify.enabled` is false, and prints swaks'
reason when the send fails (exit 1). A run whose report could not be sent says so
too: a `[notify] done FAIL` line, `report mail  FAIL` in `bseq show`, and
`mail:FAIL` in `bseq history`.
