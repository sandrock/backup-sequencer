backup-sequencer
================================

A small, dependency-light backup orchestrator written in PowerShell. It runs a
set of per-service backups in a defined time window — mandatory services first —
without overlapping, records a structured event log, and reports each run by
email. Services are described in JSON and implemented by step scripts: reusable
*recipes* or *bespoke* scripts, both following the same step contract.

It is a coordinator, not a backup format, and it is only a *producer*: it drives
your stop/dump/start steps in the right order, publishes each finished backup to a
staging area, and records what happened. Archiving — shipping a backup somewhere
durable, with or without deduplication — is a separate, pluggable concern handled
by an external archiver, not by the sequencer. See `docs/archiving.md`.


Principles
----------------------------------------------------------------

- Producer, not archiver. The sequencer builds backups and publishes them to a
  staging area; it never ships them, and the only backup it deletes is a stale,
  unconsumed one it replaces (with a notification). Archiving is an external,
  pluggable consumer — FTP, a remote pull, restic, object storage — and the
  sequencer is indifferent to the method.
- A backup is a set of artifacts. A service backup is a directory of separate
  files (e.g. config, binaries, data, a database dump), not one archive, so each
  can be restored or deduplicated on its own.
- Atomic handoff. A finished backup moves `wip → stage → ready` by atomic
  directory rename, so a consumer never sees a half-written backup.
- Bounded on disk. Staging keeps at most the latest backup per service; if the
  archiver falls behind, the old one is replaced with a notification rather than
  accumulated. Deletion is a rename into `trash/`, reaped off the hot path.


Features
----------------------------------------------------------------

- One time window, no overlap — replaces a pile of independent cron jobs.
- Mandatory services always run first.
- Structured, append-only event log (JSONL) → `plan`, `history`, and reports
  are all projections over it.
- Resource-aware step model (named locks) so independent steps can run in
  parallel later without config changes.
- Per-service failure isolation, with `alwaysRun` steps so a service is never
  left stopped after a failed backup.
- Cross-platform (Linux + Windows, PowerShell 7).


Layout
----------------------------------------------------------------

```
src/
  sequencer.ps1        entry point: plan | run | history | help
  lib/BackupCommon.psm1 engine (events, projection, locking, planning, run)
  recipes/             reusable step scripts (e.g. docker-compose-cold)
  services/            bespoke per-service step scripts (same contract)
tests/                 Pester tests + abstract fixtures
docs/                  developer specs and internals (see archiving.md)
help/                  end-user documentation
config.sample.json     example config (placeholders)
```


Quick start
----------------------------------------------------------------

```sh
# run the tests (no docker, no network needed)
pwsh -c "Invoke-Pester -Path ./tests"

# dry-run a plan against a config
pwsh src/sequencer.ps1 plan -ConfigPath ./config.sample.json

# execute (use -Force to bypass the time-window gate for a manual run)
pwsh src/sequencer.ps1 run -ConfigPath ./config.sample.json -Force
```

See `help/` for installation locations, scheduling, and notifications, and
`docs/` for the internals and how to add a recipe or service.


License
----------------------------------------------------------------

TBD.
