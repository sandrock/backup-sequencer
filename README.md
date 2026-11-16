backup-sequencer
================================

A small, dependency-light backup orchestrator in PowerShell. It runs a set of
per-service backups inside one nightly window, driven entirely by an append-only
event log, and stays out of the way of *how* each service is backed up or *where*
the result is shipped.


Why
----------------------------------------------------------------

A machine accrues a pile of independent backup cron jobs. They overlap and fight
for disk and I/O; a slow one overruns into working hours; a service is left stopped
after a failed dump; one job quietly stops running and nobody notices; and naive
"delete files older than N days" retention slowly erodes how many distinct days you
can actually restore.

The sequencer replaces that pile with one coordinator that plans each night from
what has — and hasn't — happened, and records everything. It is a *coordinator*,
not a backup format: it drives your stop/dump/start steps in order, publishes each
finished backup to a spool, runs whatever *archiver* you plug in to ship and retain
it, and logs the outcome — without baking a format, a destination, or a retention
policy into the engine. See `docs/archiving.md`.


Principles
----------------------------------------------------------------

**The event log is the single source of truth.** Every run appends structured
events (JSONL). `plan`, `history`, retention, and "is this already backed up?" are
all *projections* over that log — never over directory names or file mtimes. There
is no hidden state.

**Fluid, not cron.** Each run computes a fresh plan and packs the window with as
much backup as it can. Services are ordered by a fairness score — most overdue
first — so every service gets its turn and nothing starves; a job that will not fit
the time left is deferred to a later night, and anything that can't keep up shows as
a rising *overdue* in `plan` rather than being silently dropped. A started job is
never killed, so backups finish; the window only gates what *starts*.

**Don't do needless work.** A service already backed up for its period is skipped —
no dump, no downtime. The archiver makes that call, because only it knows its own
retention: with GFS tiers, a yearly or weekly backup also covers the day.

**Archiving is pluggable, not built in.** The engine produces each backup and
publishes it to a spool; it has no shipping method or retention policy of its own.
Copying the set to a destination and pruning old copies is an *archiver recipe* — it
can run as a step inside the run (the bundled `archive-*` recipes do this: ship to a
destination and prune by GFS / keep-N) or as an external consumer that drains
`ready/` on its own schedule. The method lives in the recipe; the engine stays
indifferent to it. The one thing the engine itself ever deletes is a stale,
unconsumed backup it replaces at publish time (with a notification).

**A backup is a set of artifacts.** A service backup is a directory of separate
files (config, binaries, data, a database dump), not one blob — so each can be
restored or deduplicated on its own.

**Atomic and bounded.** A finished backup moves `wip → stage → ready` by atomic
directory rename, so a consumer never sees a half-written set. The spool keeps at
most the latest backup per service; deletion is a rename into `trash/`, reaped off
the hot path.


What it does
----------------------------------------------------------------

- One nightly window, no overlapping runs (a single-instance lock) — in place of a
  pile of independent cron jobs.
- Log-driven `plan` / `run` / `history`, plus an email report per run that doubles
  as a dead-man's-switch (no mail means the run did not happen).
- Fluid scheduling: fairness-ordered, window-packed, coverage-aware, with tiered
  (GFS) retention that keeps N distinct periods, not N recent copies.
- Per-service failure isolation, with `alwaysRun` steps so a service is never left
  stopped after a failed backup.
- Pluggable *recipes* (reusable, e.g. `docker-compose-cold`) and *bespoke*
  per-service scripts, both on the same step contract.
- Cross-platform (Linux + Windows, PowerShell 7); no runtime dependency beyond
  `pwsh` and whatever a given recipe's steps call.


Quick start
----------------------------------------------------------------

```sh
# run the tests (no docker, no network needed)
pwsh -c "Invoke-Pester -Path ./tests"

# dry-run a plan against a config (no side effects)
pwsh src/sequencer.ps1 plan -ConfigPath ./config.sample.json

# execute; -ForceWindow runs outside the time window for a manual run,
# -ForceAllJobs also ignores the fit/coverage checks (back up everything now)
pwsh src/sequencer.ps1 run -ConfigPath ./config.sample.json -ForceWindow
```

Installed (see [Getting started](help/getting-started.md)), the same commands are
`bseq plan`, `bseq run`, `bseq history`.


Documentation
----------------------------------------------------------------

User guides (`help/`):

- [Getting started](help/getting-started.md) — install, configure, run, notify.
- [Configuring a service](help/configuring-services.md) — wiring a service to a
  recipe: `script`, `env`, `steps`, archiving, scheduling.
- [docker-compose-cold recipe](help/recipe-docker-compose-cold.md) — the bundled
  cold-backup recipe and its `DCC_*` parameters.
- [Archiving](help/archiving.md) — synchronous vs asynchronous archiving,
  `publishToReady`, and the env scope chain. Per-archiver:
  [archive-copy](help/archiving-copy.md) · [archive-gfsy](help/archiving-gfsy.md) ·
  [archive-restic](help/archiving-restic.md).

Design and internals (`docs/`):

- [Scheduling](docs/scheduling.md) — the fairness score and window packing.
- [Spool and archiving model](docs/archiving.md) — `wip → stage → ready`, the
  producer/consumer contract.
- [Architecture and internals](docs/architecture.md) — engine structure, data
  model, resource model.


Layout
----------------------------------------------------------------

```
src/
  sequencer.ps1         entry point: plan | run | history | help
  lib/BackupCommon.psm1 engine (events, projection, locking, planning, run)
  recipes/              reusable step scripts (e.g. docker-compose-cold)
  services/             bespoke per-service step scripts (same contract)
tests/                  Pester tests + abstract fixtures
docs/                   design and internals (archiving.md, scheduling.md)
help/                   end-user documentation (install, schedule, notify)
packages/               installer + systemd units
config.sample.json      example config (placeholders)
```


License
----------------------------------------------------------------

TBD.
