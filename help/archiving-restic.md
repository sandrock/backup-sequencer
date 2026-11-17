archive-restic recipe
================================

Ship each backup off the machine into a [restic](https://restic.net) repository:
deduplicated, encrypted, and with restic's own grandfather-father-son retention.
Bundled with the sequencer at [`recipes/archive-restic.ps1`](../src/recipes/archive-restic.ps1).

This is a [synchronous archiver](archiving.md) — it runs inside the backup window,
against the completed set in the service's `stage/` dir. It is the recommended
off-box archiver when you want dedup and encryption; the flat `archive-copy` and
tiered `archive-gfsy` recipes stay for plain on-disk copies (see
[Archiving](archiving.md)).


Why restic
----------------------------------------------------------------

- **Dedup.** Feed a full dump every night; restic stores only the blocks that
  changed since the last snapshot. You get the space profile of incrementals with
  no per-service incremental support — most nights are a tiny delta.
- **Encryption at rest.** The repo is encrypted before anything is written, so it
  is safe on third-party or remote storage.
- **Small footprint.** One static binary, no daemon, no database. A restore needs
  only the binary and the repo password.

Feed restic **uncompressed** input. A pre-compressed dump (`.gz`, `.7z`) defeats
dedup — one changed byte rewrites the whole stream. Let the service recipe drop its
own compression and let restic compress in-repo (`RESTIC_COMPRESSION=max`).


Configuration
----------------------------------------------------------------

Point the service's `syncArchive` at the recipe. `syncArchive` runs an archiver
**synchronously**, inside the backup window; it is a separate thing from an
**asynchronous** archiver that drains `ready/` on its own schedule. You may run
either, or both — this recipe is the synchronous one.

`publishToReady` is an independent decision about the local `ready/` copy, not
something restic dictates:

- `false` — this restic snapshot is the only archiver, so there is nothing to keep
  in `ready/` (no async consumer, no hot local copy). Drop it.
- `true` — keep the `ready/` copy as well: either you also run an async archiver
  that picks it up (synchronous restic *and* async), or you just want a hot local
  copy on hand.

A restic-only setup (no async archiver) therefore uses `false`. restic's settings
go in an `env` map; `syncArchive.env` is the archiver-only scope, which is a good
home for the password file — it stays out of the backup recipe's environment. The
service entry sits under the config's `services` object:

```json
{
  "services": {
    "example-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "publishToReady": false,
      "syncArchive": {
        "script": "recipes/archive-restic.ps1",
        "env": {
          "RESTIC_REPOSITORY": "/srv/restic/core-services",
          "RESTIC_PASSWORD_FILE": "/etc/backup-sequencer/secrets/restic-core.pwd",
          "RESTIC_COMPRESSION": "max"
        }
      },
      "steps": [ "..." ]
    }
  }
}
```

Values shared across services (a common `RESTIC_REPOSITORY`) can be set **once** in
the top-level `env` and overridden per service; the scope chain is
`top-level env ← service env ← syncArchive.env` (see
[Archiving — Environment](archiving.md#environment)):

```json
{
  "env": {
    "RESTIC_REPOSITORY": "/srv/restic/core-services",
    "RESTIC_COMPRESSION": "max"
  },
  "services": { "example-app": { "...": "repo inherited; keep RESTIC_PASSWORD_FILE in syncArchive.env" } }
}
```

Environment:

| Variable | Required | Meaning |
|---|---|---|
| `RESTIC_REPOSITORY` | yes | Repo location: a path, or a backend URL (`sftp:…`, `rclone:…`, `s3:…`). |
| `RESTIC_PASSWORD_FILE` | — | File with the repo password. Or set `RESTIC_PASSWORD`. |
| `RESTIC_COMPRESSION` | — | `max` / `auto` / `off`. Feed uncompressed input and use `max`. |
| `RESTIC_HOST` | — | Snapshot host tag. Defaults to the machine's hostname. |
| `RESTIC_TAGS` | — | Extra tags, `;`-separated. The service name is always a tag. |
| `RESTIC_MIN_INTERVAL_HOURS` | — | Coverage interval for the overdue query (default 24). |
| `RESTIC_BIN` | — | restic executable (default `restic`). |

Any other `RESTIC_*` (and backend variables restic reads from the environment) is
passed through to the restic process, so remote backends work without changing the
recipe. Keep passwords in `secrets/` (mode 0600, never committed), not in
`config.json`.

Each run produces **one snapshot**, tagged with the service name and host:

```
cd <stage>/<service> && restic backup . --tag <service> --host <host>
```

restic runs from **inside** the stage dir and backs up `.`, so the snapshot holds the
set's files at its root (`/<service>_data.tar`, …) rather than under the spool path.
The snapshot's recorded path is still the absolute stage dir, which restic uses to
find the previous snapshot to compare against: if the spool moves, the next backup
of each service re-reads its files once (nothing is stored twice — dedup still
applies).

Query a service's history by metadata, not by folder:
`restic snapshots --tag example-app --host <host>`. The sequencer records the new
snapshot id in its own log (`service.archived`), so a run ties back to its snapshot.


Retention (GFS) runs out of band
----------------------------------------------------------------

The recipe **only ever adds snapshots** — it never prunes. restic's retention is a
separate, exclusive operation, so run it on its own schedule, off the backup
window (never while a backup might be writing):

```sh
restic forget --group-by host,tags \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 12 --keep-yearly 3 --prune
```

- `--group-by host,tags` is important: it applies the policy **per service**, since
  one repo holds every service's snapshots.
- `--prune` actually frees space and is exclusive — schedule it apart from backups,
  and run `--dry-run` first.

Keeping retention outside the run means a slow or failed prune can never delete a
backup mid-window.


Coverage (don't re-snapshot)
----------------------------------------------------------------

The recipe reports *overdue* to the planner from the event log: a service whose
newest snapshot is younger than `RESTIC_MIN_INTERVAL_HOURS` (default 24) is
**covered** and skipped that window — no dump, no downtime. A failed snapshot
writes no `service.archived` event, so the service stays overdue and the next
window retries it automatically. See [the scheduling model](../docs/scheduling.md).


Restore
----------------------------------------------------------------

```sh
restic -r <repo> snapshots --tag <service> --host <host>
restic -r <repo> restore latest --tag <service> --host <host> --target /restore/<service>
restic -r <repo> mount /mnt/browse     # or browse snapshots as files
```

The set's files land directly in the target (`/restore/<service>/<service>_data.tar`,
…). Snapshots taken by versions before this layout hold the full stage path instead;
restore just their folder to get the same flat result:

```sh
restic -r <repo> restore <id>:/var/spool/backup-sequencer/stage/<service> --target /restore/<service>
```

Then run the service's own restore step (extract / import / verify). A restore
needs the restic binary, the repo password, and reachability to the repo — keep
those recorded somewhere off every backed-up machine.


Replication and a second copy
----------------------------------------------------------------

A single repo is a single point of failure. restic can copy snapshots between
repos (`restic copy --from-repo …`), so a second machine can pull the repo and
keep a longer retention ladder than the primary. That topology (where the repos
live, which host pulls, the schedules) is a deployment decision and lives with
your infrastructure, not in this tool.
