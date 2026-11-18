archive-copy recipe
================================

Copy each backup to a destination and keep the newest N — a flat history, no
tiers. Bundled with the sequencer at
[`recipes/archive-copy.ps1`](../src/recipes/archive-copy.ps1).

This is a [synchronous archiver](archiving.md): it runs inside the backup window,
against the completed set in the service's `stage/` dir. It is the simplest
archiver — reach for it when you just want a few recent copies on another disk or
mount. For tiered retention use [`archive-gfsy`](archiving-gfsy.md); for an off-box,
deduplicated, encrypted repository use [`archive-restic`](archiving-restic.md).


What it does
----------------------------------------------------------------

Each run copies the complete artifact set into a directory named by the run id:

```
<ARCHIVE_DEST>/<service>/<runId>/<artifacts>
```

Then it prunes that service to the newest `ARCHIVE_KEEP` copies, printing each
path it deletes with its size. A copy it fails to delete stays recorded as live and
is pruned again on the next run (a warning, not a failure). Naming each copy
by the run id ties an archive directly back to the run that produced it (and to its
`service.archived` event).


Configuration
----------------------------------------------------------------

Point the service's `syncArchive` at the recipe. Set `publishToReady: false` when
archive-copy is the only archiver — the copy under `ARCHIVE_DEST` is the archive,
so no separate copy need be kept in `ready/`. See [Archiving](archiving.md) for how
the two archiver shapes and `publishToReady` fit together.

The recipe reads its settings from the service's `env`, resolved along the scope
chain `top-level env ← service env ← syncArchive.env` (see
[Archiving — Environment](archiving.md#environment)). Set `ARCHIVE_DEST` **once** in
the top-level `env` so all services write under the same destination, and keep only
per-service tuning (like `ARCHIVE_KEEP`) in each service's `env`:

```json
{
  "env": { "ARCHIVE_DEST": "/var/backups/cloud" },
  "services": {
    "example-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "env": { "ARCHIVE_KEEP": "7" },
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-copy.ps1" },
      "steps": [ "..." ]
    },
    "other-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-copy.ps1" },
      "steps": [ "..." ]
    }
  }
}
```

Both services inherit `ARCHIVE_DEST` from the top-level `env` (each writing under
`<ARCHIVE_DEST>/<service>/`); `example-app` keeps 7 copies while `other-app` takes
the default. A service can point itself elsewhere by setting `ARCHIVE_DEST` in its
own `env`.

| Variable | Default | Meaning |
|---|---|---|
| `ARCHIVE_DEST` | *(required)* | Destination base dir; the service name is appended. |
| `ARCHIVE_KEEP` | `3` | How many copies to keep. |


Pruning (log-driven)
----------------------------------------------------------------

Pruning keeps the newest `ARCHIVE_KEEP` archives and deletes the rest — but it
decides *entirely from the event log*, never from directory names or mtimes. The
recipe only ever deletes paths it previously recorded as archived, so a foreign
file dropped into the destination is never at risk. This is correct even under the
sequencer's irregular, non-cron timing.


Coverage (don't re-back-up)
----------------------------------------------------------------

A service whose newest copy is younger than its `targetIntervalHours` (default 24,
set on the service) is **covered** and skipped that window — no dump, no downtime.
The engine reads that from the event log. See [the scheduling model](../docs/scheduling.md).


Restore
----------------------------------------------------------------

The archive is a plain directory copy — restore by copying a run's set back and
running the service's own restore step (extract / import / verify):

```sh
ls <ARCHIVE_DEST>/<service>/           # pick a run id
cp -a <ARCHIVE_DEST>/<service>/<runId>/. /restore/<service>/
```
