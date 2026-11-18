archive-gfsy recipe
================================

Keep a tiered grandfather-father-son history on disk — daily, weekly, and yearly
copies — routed and pruned from the event log. Bundled with the sequencer at
[`recipes/archive-gfsy.ps1`](../src/recipes/archive-gfsy.ps1).

This is a [synchronous archiver](archiving.md): it runs inside the backup window,
against the completed set in the service's `stage/` dir. Use it when you want a few
recent days plus longer-spaced weekly and yearly restore points on a local disk or
mount. For a single flat history use [`archive-copy`](archiving-copy.md); for an
off-box, deduplicated, encrypted repository use
[`archive-restic`](archiving-restic.md).


How it routes a run to a tier
----------------------------------------------------------------

Each run lands in exactly **one** tier, decided from the log (coarsest empty slot
wins):

1. **yearly** — if the log shows no yearly archive this calendar year;
2. else **weekly** — if none this ISO week;
3. else **daily**.

So the first backup of a year is the yearly, the first of each later week is a
weekly, and the rest of the week are dailies. The copy lands in a per-tier
directory named by the run id:

```
<ARCHIVE_DEST>/<service>/<tier>/<runId>/<artifacts>
```


Retention (per period, not per copy)
----------------------------------------------------------------

Each tier keeps **the newest archive of each of the last N periods** — not the N
most recent copies. A tier is pruned only on the runs that land in it. Because
retention is per period, extra same-period runs collapse to one kept archive, so
they can never erode how many distinct days / weeks / years you retain. Like the
routing, pruning reads the recorded timestamps and paths from the log, never
directory names — and only ever deletes paths the recipe itself recorded, so
foreign files are safe. Each deletion is printed with its size; one that fails
stays recorded as live and is pruned again on the next run (a warning).


Configuration
----------------------------------------------------------------

Point the service's `syncArchive` at the recipe. Set `publishToReady: false` when
archive-gfsy is the only archiver — the tiered copies under `ARCHIVE_DEST` are the
archive, so no separate copy need be kept in `ready/`. See
[Archiving](archiving.md) for how the archiver shapes and `publishToReady` fit
together.

The recipe reads its settings from the service's `env`, resolved along the scope
chain `top-level env ← service env ← syncArchive.env` (see
[Archiving — Environment](archiving.md#environment)). Set `ARCHIVE_DEST` **once** in
the top-level `env` so all services write under the same destination, and keep only
per-service tuning (the keep counts) in each service's `env`:

```json
{
  "env": { "ARCHIVE_DEST": "/var/backups/cloud" },
  "services": {
    "example-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "env": { "ARCHIVE_KEEP_DAILY": "7", "ARCHIVE_KEEP_WEEKLY": "4", "ARCHIVE_KEEP_YEARLY": "3" },
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-gfsy.ps1" },
      "steps": [ "..." ]
    },
    "other-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-gfsy.ps1" },
      "steps": [ "..." ]
    }
  }
}
```

Both services inherit `ARCHIVE_DEST` from the top-level `env` (each writing under
`<ARCHIVE_DEST>/<service>/<tier>/`); `example-app` tunes its keep counts while
`other-app` takes the defaults (2 each). A service can point itself elsewhere by
setting `ARCHIVE_DEST` in its own `env`.

| Variable | Default | Meaning |
|---|---|---|
| `ARCHIVE_DEST` | *(required)* | Destination base dir; the service name is appended. |
| `ARCHIVE_KEEP_DAILY` | `2` | Daily periods to keep. |
| `ARCHIVE_KEEP_WEEKLY` | `2` | Weekly periods to keep. |
| `ARCHIVE_KEEP_YEARLY` | `2` | Yearly periods to keep. |


Coverage (cross-tier)
----------------------------------------------------------------

Coverage is **cross-tier**: a service is covered if *any* tier was archived within
its `targetIntervalHours` (default 24, set on the service) — a yearly or weekly
backup also covers the daily. So a covered service is skipped that window (no dump, no downtime), and the
day's single backup is routed to whichever tier is due. See
[the scheduling model](../docs/scheduling.md).


Restore
----------------------------------------------------------------

Each archive is a plain directory copy. Pick a tier and run id, copy the set back,
and run the service's own restore step (extract / import / verify):

```sh
ls <ARCHIVE_DEST>/<service>/            # daily / weekly / yearly
ls <ARCHIVE_DEST>/<service>/weekly/     # pick a run id
cp -a <ARCHIVE_DEST>/<service>/weekly/<runId>/. /restore/<service>/
```
