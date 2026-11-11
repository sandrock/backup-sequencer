backup-sequencer
================================

A small, dependency-light backup orchestrator written in PowerShell. It runs a
set of per-service backups in a defined time window — mandatory services first —
without overlapping, records a structured event log, and reports each run by
email. Services are described in JSON and implemented by step scripts: reusable
*recipes* or *bespoke* scripts, both following the same step contract.

It is a coordinator, not a backup format: it drives your dump/stop/start/ship
steps in the right order and keeps a history of what happened. Shipping and
deduplication (e.g. to a remote repository) are provided as steps, not built in.


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
docs/                  developer specs and internals
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
