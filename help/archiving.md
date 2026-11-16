Archiving
================================

How a finished backup gets shipped somewhere durable and pruned over time.

The sequencer is a *producer*: it runs your stop/dump/start steps, assembles each
backup as a set of files, and publishes it to a local spool. It never ships a
backup off the machine or decides a retention policy itself — that is an
*archiver's* job. This keeps the engine free of any particular destination or
format, and lets you pick (or write) the archiver that fits.

There are two archiver shapes, and you can use **either, or both**:

- **Synchronous** — runs *inside* the backup window, as a step of the run.
- **Asynchronous** — a separate process that drains the spool on its *own*
  schedule, outside the sequencer.

They are independent choices, tied together by one config key, `publishToReady`
(whether a copy is left in the spool's `ready/` dir for an async archiver):

| Setup | `syncArchive` | `publishToReady` |
|---|---|---|
| Synchronous only | set | `false` |
| Asynchronous only | — | `true` |
| Both | set | `true` |

For the internals behind all of this — the spool phases, atomic promotion, and the
`ready/` contract — see [the spool and archiving model](../docs/archiving.md).


Synchronous archiving
----------------------------------------------------------------

A synchronous archiver runs *during* the run, as the service's `syncArchive` step,
against the completed backup in `stage/<service>/`. Because it runs in-window, it
is simplest for fast ships (a local copy, an FTP push, a restic snapshot over a
fast link); a slow ship eats into the window for the services after it.

Configure it per service with `syncArchive` (just the archiver `script`). Its
settings come from the service's `env`, so a value can be shared with the backup
recipe and set once globally — see [Environment](#environment) below. Set
`publishToReady: false` when this is the only archiver — the ship *is* the archive,
so no copy need be kept in `ready/` (keeping one nobody fetches would only pile up
disk):

```json
{
  "env": { "ARCHIVE_DEST": "/var/backups/cloud" },
  "services": {
    "example-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "env": { "DCC_COMPOSE_DIR": "/srv/example-app" },
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-gfsy.ps1" },
      "steps": [ "..." ]
    }
  }
}
```

Three archivers ship with the tool; select one per service via
`syncArchive.script`, and see each page for its env and layout:

- **[`archive-copy`](archiving-copy.md)** — copy the set to a destination, keep the
  newest N. A flat history, simplest to reason about.
- **[`archive-gfsy`](archiving-gfsy.md)** — a tiered grandfather-father-son history
  on disk (daily / weekly / yearly), routed and pruned from the event log.
- **[`archive-restic`](archiving-restic.md)** — snapshot the set into an off-box
  [restic](https://restic.net) repository: deduplicated, encrypted, with restic's
  own retention.

If a synchronous archiver fails, it is reported (the run email flags it). Because a
failed archive records nothing, coverage leaves the service overdue so the next
window retries it — and with `publishToReady: false` the just-made backup is simply
dropped (nothing retained, nothing silently stranded). The failure is never silent.


Asynchronous archiving
----------------------------------------------------------------

An asynchronous archiver is a separate process — local or on another host — that
the sequencer does **not** run. It watches the spool's `ready/` directory, takes
each completed `ready/<service>/`, ships it, and deletes it, all on its own
schedule. Use it when the ship is slow or heavy (a large transfer, a remote pull
over the internet): it runs off the backup window, so it never competes with the
night's backups.

On the sequencer side there is almost nothing to configure — you do **not** list
an async archiver in `config.json`. You only need a copy to appear in `ready/`, so
set (or leave) `publishToReady: true` — the default — and omit `syncArchive`:

```json
{
  "services": {
    "example-app": {
      "script": "recipes/docker-compose-cold.ps1",
      "publishToReady": true,
      "steps": [ "..." ]
    }
  }
}
```

After each run, the completed backup is at
`<spool>/ready/<service>/` (the spool root is the `spool` config key, default
`/var/spool/backup-sequencer`). The sequencer keeps `ready/` bounded to the latest
backup per service; your archiver's job is to take it from there.

The archiver itself is anything that reads a directory — `rsync`, `scp`, `restic`
run from the far side, an object-storage upload, tape — on its own cron or timer.
Two ways to keep it from reading a set while the sequencer replaces it:

- **Schedule apart** (simple) — run the archiver outside the backup window.
  Producer and consumer never touch `ready/<service>/` at the same time, so a plain
  copy-then-delete is safe. For example, a cron on the archive host, well after the
  window closes:
  ```sh
  rsync -a --remove-source-files \
    backup@prod:/var/spool/backup-sequencer/ready/ /archive/incoming/
  ```
- **Claim by rename** (overlap-safe) — the archiver first moves `ready/<service>/`
  aside with one atomic rename to claim it, then ships at leisure, then deletes.
  Safe even if it runs while the sequencer does.

Because the async archiver owns deletion, it — not the sequencer — enforces how
many copies are kept at the destination.


Environment
----------------------------------------------------------------

Recipes and archivers read their settings (`DCC_*`, `ARCHIVE_*`, `RESTIC_*`, …)
from `env` maps, resolved along a scope chain — later layers override earlier ones:

- **`env`** (top-level) — shared by every service. Set a value once here: a common
  `ARCHIVE_DEST`, a shared `RESTIC_REPOSITORY`.
- **`env`** (per-service) — this service's settings, for its backup recipe *and* its
  archiver.
- **`syncArchive.env`** (optional) — archiver-only overrides, for a value you want
  scoped to the archiver and kept out of the backup recipe (e.g. a repo password
  file).

The backup recipe sees `top-level ← service`; the archiver sees
`top-level ← service ← syncArchive`. Distinct prefixes (`DCC_`, `ARCHIVE_`,
`RESTIC_`) keep the namespaces from colliding, so mixing them in one `env` is fine.
Keep secrets in `secrets/` (mode 0600) and reference them by path.


Writing your own archiver
----------------------------------------------------------------

Either shape can be custom. A synchronous one is a recipe exposing an `archive`
step (model it on [`archive-copy`](archiving-copy.md)); an asynchronous one is just
a program that drains `ready/`. The contract for both — what `ready/<service>/`
guarantees, the atomic promotion, deletion via `trash/` — is in
[the spool and archiving model](../docs/archiving.md).
