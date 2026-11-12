Architecture and internals
================================

Developer-facing design of the sequencer. For usage see `../help/`.

The spool and archiving handoff is specified in `archiving.md` (producer/consumer
split, the `wip → stage → ready` lifecycle) and is implemented. This file covers
the engine's structure; see `archiving.md` for the spool model in depth.


Components
----------------------------------------------------------------

- `src/sequencer.ps1` — thin entry point. Resolves config/state paths, loads the
  config, dispatches `plan` / `run` / `history`, and (on `run`) renders + sends
  the report.
- `src/lib/BackupCommon.psm1` — the engine. Pure-ish helpers plus the run loop.
- `src/recipes/*.ps1` — reusable step scripts parameterised by environment.
- `src/services/*.ps1` — bespoke per-service step scripts (same contract as a
  recipe; for services that don't fit a reusable pattern).


The step contract
----------------------------------------------------------------

A recipe or service script is invoked once per step:

```
<script>.ps1 -Step <name> -OutDir <path> -Service <name>
```

It performs exactly that step and throws on failure. Per-service parameters are
passed as environment variables, declared in the service's `env` block in the
config — this is what lets one recipe serve many services, and the same script
target different hosts.


Data model
----------------------------------------------------------------

Two files, single-writer each:

- `config.json` — admin-owned, declarative. Window, named resources, notify
  settings, and a `services` map. Each service lists ordered `steps`; each step
  declares the `resources` it holds and optional `retries` / `timeoutSec` /
  `alwaysRun`.
- `events.jsonl` — engine-owned, append-only. One typed line per event
  (`run.start`, `run.plan`, `step.start`, `step.end`, `service.end`,
  `service.skip`, `service.replaced`, `service.syncfail`, `run.windowclosed`,
  `run.end`). Crash-safe and the single source of truth.

"Current state" is a projection folded from the event log
(`Get-ServiceProjection`): per service, last success, typical step durations,
last size in bytes, consecutive failures. Planning, the pre-flight space check,
and reports all read this projection — nothing else is persisted.


Run lifecycle
----------------------------------------------------------------

1. Take the single-instance lock (a second run exits immediately).
2. Reap `trash/` (off the publish path); emit `run.start`; fold the projection;
   compute the plan; emit `run.plan`.
3. For each planned service, while inside the window (unless `-Force`):
   - pre-flight free-space check against the last recorded size;
   - apply the service's `env`; run each step via `Invoke-BackupStep` (timing,
     retries, events), producing the artifact set into `wip/<service>/`;
   - after a failure, skip normal steps but still run `alwaysRun` steps (e.g.
     `start`) so the service is never left down, then move the partial to `trash/`;
   - on success, promote `wip → stage`, run the optional sync-archive step, then
     publish `stage → ready` (replacing + notifying if the previous was unconsumed);
   - record the produced size, emit `service.end`.
4. Emit `run.end`; release the lock. The entry point then renders the report and
   sends it if notifications are enabled.


Planning
----------------------------------------------------------------

`Get-BackupPlan` is a pure function of `(config, projection, time)`. It returns
mandatory services first (in config order), then the rest (in config order).
Because it is pure, `plan` has no side effects and the logic is unit-testable.
Smarter tail selection (fitting estimated durations to the remaining window) can
replace the ordering later; the projection already carries the data for it.


Resource model
----------------------------------------------------------------

Each step lists named resources (e.g. `diskA`, `net-out`), each a lock with a
capacity (default 1). Steps sharing a resource cannot overlap; steps on disjoint
resources can. The current executor runs strictly sequentially; the declarations
exist so a parallel executor can be added without touching any config.


Adding a recipe or a service
----------------------------------------------------------------

- Recipe: add `src/recipes/<name>.ps1` implementing the step contract, reading
  its parameters from environment variables. Point a service's `script` at it and
  supply the variables in the service's `env`.
- Bespoke: add `src/services/<name>.ps1` with the same contract when no reusable
  pattern fits. Oddballs are first-class — they just skip the shared recipe.
