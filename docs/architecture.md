Architecture and internals
================================

Developer-facing design of the sequencer. For usage see the
[help guides](../help/getting-started.md).

The spool and archiving handoff is specified in [the spool and archiving
model](archiving.md) (producer/consumer split, the `wip → stage → ready`
lifecycle) and is implemented. This file covers the engine's structure; see that
doc for the spool model in depth.


Components
----------------------------------------------------------------

- `src/sequencer.ps1` — thin entry point. Resolves config/state paths, loads the
  config, dispatches `plan` / `run` / `status` / `show` / `history`, and (on `run`) renders + sends
  the report.
- `src/lib/BackupCommon.psm1` — the engine. Pure-ish helpers plus the run loop.
- `src/recipes/*.ps1` — reusable step scripts parameterised by environment.
- `src/services/*.ps1` — bespoke per-service step scripts (same contract as a
  recipe; for services that don't fit a reusable pattern).


The step contract
----------------------------------------------------------------

A recipe or service script takes a single parameter, a `RecipeContext`, and uses it
to register its steps:

```
param($Recipe)
$Recipe.AddStep('dump', { param($ctx) ... })   # register a step
$Recipe.SetOverdue({ param($ctx) ... })        # optional: archiver coverage query
$Recipe.SetValidate({ param($ctx) ... })       # optional: preflight prereq check
```

The engine loads the script once per service (which only *registers* steps), then
runs each step listed in the service's `steps`, in order, by invoking that step's
block with a shared context `$ctx`. A step throws on failure.

The context carries everything a step needs:

- `$ctx.Service` / `$ctx.RunId` / `$ctx.Now` — the service name, run id, reference time.
- `$ctx.OutDir` — where to write: the service's `wip/` dir for a backup step, its
  `stage/` dir for the archive step.
- `$ctx.Env` — the merged env, read via `$ctx.EnvValue(name, default)` /
  `$ctx.RequireEnv(name)`. Per-service parameters come from `env`, which is what lets
  one recipe serve many services and target different hosts.
- `$ctx.State` — a hashtable shared across this recipe's steps (e.g. `WasRunning`, so
  `start` only restarts what `stop` took down).
- `$ctx.EventsPath` / `$ctx.ReadEvents()` — read-only access to the event log, for
  archivers that route / prune / compute coverage from it.
- `$ctx.ConfiguredSteps` — the step names the service config lists, in run order
  (backup recipe only). Lets a step adapt to its siblings and a validate block check
  that steps meant to go together are configured together (e.g. `dump` leaves hot
  dirs out only when `hot-dump` is listed).

An archiver recipe additionally registers `SetOverdue` (its plan-time coverage
query) and returns its result (tier / path / snapshot, pruned paths) via
`$ctx.State['Result']`, which the engine records as `service.archived`.

A recipe (backup or archiver) may also register `SetValidate`, returning a list of
problem strings (empty = OK). The engine runs it — plus structural checks (recipe /
archiver scripts resolve, every configured step is registered, the archiver has an
`archive` step) — as a **preflight before any step runs**: a service that fails is
skipped (`service.end status=fail reason=invalid`) without being taken down, so a
doomed archive (missing password file, unreachable destination) never costs it its
downtime. The same checks back the `validate` command, which runs nothing. Validation
sits after the coverage/fit gates in a run, so it only vets services about to run;
`validate` vets every enabled service.

Env resolves along a scope chain, later overriding earlier: a backup recipe sees
`config.env ← service.env`; an archiver also sees `← syncArchive.env`.


Data model
----------------------------------------------------------------

Two files, single-writer each:

- `config.json` — admin-owned, declarative. Window, named resources, notify
  settings, a shared `env`, and a `services` map. Each service lists ordered
  `steps`; a step may set `retries` and `alwaysRun` (both honored), plus the
  `resources` it holds and a `timeoutSec` — the latter two are declared but not yet
  enforced (see Resource model).
- `events.jsonl` — engine-owned, append-only. One typed line per event
  (`run.start`, `run.plan`, `step.start`, `step.end`, `service.end`,
  `service.skip`, `service.replaced`, `service.syncfail`, `service.archived`,
  `service.archive-pruned`, `run.windowclosed`, `run.end`). Crash-safe and the
  single source of truth — e.g. archivers route and prune by `service.archived` /
  `service.archive-pruned` rather than by directory names.

"Current state" is a projection folded from the event log
(`Get-ServiceProjection`): per service, last success, typical step durations,
last size in bytes, consecutive failures. Planning, the pre-flight space check,
and reports all read this projection — nothing else is persisted.


Run lifecycle
----------------------------------------------------------------

1. Take the single-instance lock (a second run exits immediately).
2. Reap `trash/` (off the publish path); emit `run.start`; fold the projection;
   compute the plan; emit `run.plan`.
3. For each planned service, while inside the window (unless `-ForceWindow`) and it
   fits the time left (unless mandatory or `-ForceAllJobs`; see [scheduling.md](scheduling.md)):
   - pre-flight free-space check against the last recorded size;
   - build the service's env (the scope chain), load the recipe (registering its
     steps), then run each listed step via `Invoke-BackupStep` (timing, retries,
     events), producing the artifact set into `wip/<service>/`;
   - after a failure, skip normal steps but still run `alwaysRun` steps (e.g.
     `start`) so the service is never left down, then move the partial to `trash/`;
   - on success, promote `wip → stage`, run the optional sync-archive step, then
     publish `stage → ready` (replacing + notifying if the previous was unconsumed);
   - record the produced size, emit `service.end`.
4. Emit `run.end`; release the lock. The entry point then renders the report and
   sends it if notifications are enabled.


Planning
----------------------------------------------------------------

`Get-BackupPlan` is a pure function of `(config, projection, overdue-by-service,
time)`. It scores every enabled service — `overdue` (age / target interval, or the
archiver's own coverage) plus a small size nudge — and returns them most-overdue
first, with mandatory services forced to the front. The run loop then packs the
window: it skips a service already covered for its period, runs what fits the time
left, and guarantees at least one job per window so progress is never zero. Being
pure, `plan` has no side effects and the scoring is unit-testable. See
[scheduling.md](scheduling.md) for the score and the window-packing rules.


Resource model
----------------------------------------------------------------

Each step lists named resources (e.g. `diskA`, `net-out`), each a lock with a
capacity (default 1). Steps sharing a resource cannot overlap; steps on disjoint
resources can. The current executor runs strictly sequentially; the declarations
exist so a parallel executor can be added without touching any config. In the same
way, a step's `timeoutSec` is recorded but not enforced — hard per-step timeouts
are a later iteration.


Adding a recipe or a service
----------------------------------------------------------------

- Recipe: add `src/recipes/<name>.ps1` taking `param($Recipe)`; register its steps
  with `$Recipe.AddStep` and read parameters from `$ctx` (the env scope chain).
  Point a service's `script` at it and supply the variables in the service's `env`.
- Bespoke: add `src/services/<name>.ps1` with the same contract when no reusable
  pattern fits. Oddballs are first-class — they just skip the shared recipe.
