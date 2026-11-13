Fluid scheduling
================================

How the sequencer decides what to back up, in what order, within one window — from
the event log, not a fixed list. The goal: use the window to do as much backup as
possible, and make sure every service gets its turn over time.

Status: implemented. The planner scores from the projection; the run loop packs the
window with a per-service fit check and a one-job-per-window guarantee.


The problem
----------------------------------------------------------------

The runner is sequential and the window is finite. The total nightly work can
exceed the window (e.g. two multi-hour services plus several small ones in a 7h
window). A fixed order either starves the small services (big ones first fill the
window) or forces overruns. We want the schedule to adapt each night from what has
and hasn't run recently.


The score
----------------------------------------------------------------

`Get-BackupPlan` returns *all* enabled services, sorted by a score computed per
service from the projection (last-success time + estimated duration):

```
age       = now - lastSuccess          # never-run -> treated as very overdue
est       = sum of last step durations # never-run -> unknown (0)
target    = desired interval           # <service>.targetIntervalHours, else
                                        # <config>.targetIntervalHours, else 24h

overdue   = age / target               # 1.0 = exactly due, 3.0 = 3x overdue
sizeNudge = est / windowLength         # 0..~1
score     = overdue + sizeNudge        # sort descending
```

- `overdue` dominates (whole-interval units), so the most-neglected service wins.
  This is the fairness engine: a service that is skipped ages, so it ranks higher
  next time — rotation is self-correcting and nothing starves.
- `sizeNudge` is a sub-one tie-breaker: among similar ages the bigger job goes
  first (it needs the most room to fit, so give it a head start).
- Never-run services float to the top (a large sentinel `overdue`) to get a first
  baseline fast.
- Mandatory services are forced to the front (still score-ordered among themselves)
  and bypass the fit check below — for a must-run-every-window service.

The generic `overdue = age/target` is a fallback. When a service has an archiver,
**the archive recipe computes `overdue` instead** (via `SetOverdue`) — only it knows
its retention periods and cross-tier coverage. `archive-gfsy` reports the service
covered (`overdue < 1`) if *any* tier was archived within the daily period, so a
yearly or weekly backup also covers the daily. `archive-copy` uses a flat
`ARCHIVE_MIN_INTERVAL_HOURS`.


Coverage (don't re-back-up)
----------------------------------------------------------------

Before the fit check, a service whose `overdue < 1` is **covered** — it already has a
fresh-enough backup for its period — and is skipped (`service.skip reason=covered`),
producing no backup and no downtime. This is what stops multiple windows or extra
triggers in one day from making many same-day copies (which, with count-based
pruning, would erode the number of distinct backup *days* you keep). `-ForceAllJobs`
overrides it. Pairs with the period-aware prune (see archiving.md): even a forced
extra same-period run collapses to one kept archive per period.


Window packing (execution)
----------------------------------------------------------------

The run loop walks the scored plan. Before starting each service, with
`remaining = seconds until close`:

1. Past close → stop (`-ForceWindow` bypasses this; a job already running is never
   killed and may overrun).
2. `overdue < 1` → **covered**, skip (`service.skip reason=covered`); `-ForceAllJobs`
   overrides.
3. Mandatory, or `-ForceAllJobs` → run (no fit check).
4. Estimate unknown (never run) → run optimistically — first backup, learns its
   duration for next time; it may overrun a lot (accepted).
5. `estimate <= remaining` → run (it fits).
6. Otherwise it does not fit. Run it **only if no non-mandatory job has run yet this
   window** (the guaranteed job); else defer it (`service.skip reason=deferred`).

Rule 5 is the key. The fit check (`estimate <= remaining`) is strict — no grace; the
window is the window. But the one-job-per-window guarantee means progress is never
zero, and it is what lets an over-window job (one whose estimate exceeds the whole
window, so it can never "fit") run: it is deferred until it becomes the most overdue,
then takes the guaranteed slot and solos the night. Because it runs first,
its overrun is bounded to `estimate - windowLength`.

`-ForceWindow` (run outside the window) and `-ForceAllJobs` (attempt every job, no
deferral) are separate switches — combine them for an unconditional "back up
everything now".

A job may still run longer than its estimate — the estimate only gates *starting*. A
slow job eats into the time left for later jobs, which then defer. Accepted.


Properties
----------------------------------------------------------------

- **Fairness / rotation.** Every service gets its turn: deferred -> older -> higher
  score -> runs. Two big services that can't both fit alternate nights; small ones
  run most nights.
- **Guaranteed progress.** At least one non-mandatory job runs per window (the most
  overdue), even if nothing fits.
- **Over-window jobs run.** They take periodic solo nights when they top the score;
  they are never deferred into permanent starvation.
- **Over-capacity is visible, not hidden.** If the total heavy demand genuinely
  exceeds capacity, no ordering can fit it — ages climb without bound. `plan` shows
  each service's `overdue xN` and `score`, so a steadily growing age is the signal to
  widen the window, parallelise, or split the job. The scheduler still does the best
  possible thing meanwhile: each night it runs the hungriest service.


Caveats
----------------------------------------------------------------

- First-ever run of a service has no estimate, so it runs optimistically and may
  overrun; subsequent runs are fit-aware.
- An over-window job that is a *cold* backup is down past the window every time it
  runs — that is a property of the job's method (prefer a hot/online backup, or split
  it), independent of scheduling.
- Estimates come from the last run's step durations; a service that grows over time
  carries a stale (smaller) estimate for one run, then corrects.
