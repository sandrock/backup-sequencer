Configuring a service
================================

Each entry in `config.json`'s `services` map wires one service to a backup
**recipe**, composes that recipe's steps into a run order, and optionally picks an
**archiver**. This page covers configuring the bundled recipes — it does not cover
writing a new one.

A recipe (e.g. [`docker-compose-cold`](recipe-docker-compose-cold.md)) is a script
that *registers* a menu of steps (`stop`, `dump`, `start`, …). Your service entry
selects which of those steps run, in what order, and passes the recipe its
parameters through `env`.


A service entry
----------------------------------------------------------------

```json
{
  "env": { "ARCHIVE_DEST": "/var/backups/cloud" },
  "services": {
    "vaultwarden": {
      "enabled": true,
      "mandatory": true,
      "script": "recipes/docker-compose-cold.ps1",
      "env": {
        "DCC_COMPOSE_DIR": "/services/vaultwarden",
        "DCC_DATA_DIRS": "/services/vaultwarden/data",
        "DCC_COMPOSE_EXCLUDES": "logs;cache"
      },
      "publishToReady": false,
      "syncArchive": { "script": "recipes/archive-gfsy.ps1" },
      "steps": [
        { "name": "stop",  "resources": [] },
        { "name": "dump",  "resources": ["diskA"] },
        { "name": "start", "resources": [], "alwaysRun": true }
      ]
    }
  }
}
```


Fields
----------------------------------------------------------------

| Field | Meaning |
|---|---|
| `enabled` | Include the service in planning. Omit or `false` to park it. |
| `mandatory` | Force it to the front of the plan and skip the window fit-check — for a must-run service. (Still skipped if already covered for its period.) |
| `script` | Path to the recipe (`recipes/<name>.ps1`) or a bespoke service script (`services/<name>.ps1`), relative to the install dir. |
| `env` | Parameters for this service's recipe and archiver (see [Environment](#environment)). |
| `steps` | The ordered run list — which of the recipe's registered steps to run (below). |
| `publishToReady` | Keep a local copy in `ready/` after the run. See [Archiving](archiving.md). |
| `syncArchive` | The archiver to run in-window (below). |
| `targetIntervalHours` | Desired backup interval for scheduling (below). |


Steps
----------------------------------------------------------------

`steps` is an explicit, ordered list — the engine runs exactly these, in this
order, and **only** these. A recipe never auto-runs its steps; if `steps` is
missing the service fails, and an empty list produces an empty backup. Each entry:

| Key | Meaning |
|---|---|
| `name` | The recipe step to run (must be one the recipe registered). |
| `alwaysRun` | Run even after an earlier step failed — use it for recovery (e.g. `start`), so a service is never left stopped. |
| `retries` | Retry the step this many times before failing it. |
| `resources` | Named resources the step holds (below). |
| `timeoutSec` | Intended per-step timeout (below). |

For `docker-compose-cold` the usual list is `stop` → `dump` → `start`, with
`start` marked `alwaysRun` so the stack comes back even if `dump` failed. See the
[docker-compose-cold recipe](recipe-docker-compose-cold.md) for what each step
does and the `DCC_*` parameters it reads.


Environment
----------------------------------------------------------------

A recipe reads its parameters (`DCC_*`, and an archiver's `ARCHIVE_*` / `RESTIC_*`)
from `env`, resolved along a scope chain — later layers override earlier:

- **`env`** (top-level) — shared by every service; set a common value once (e.g.
  `ARCHIVE_DEST`).
- **`env`** (per-service) — this service's parameters, for its recipe *and* its
  archiver.
- **`syncArchive.env`** (optional) — archiver-only overrides.

So the backup recipe sees `top-level ← service`, and the archiver sees
`top-level ← service ← syncArchive`. Full detail, with examples, is in
[Archiving — Environment](archiving.md#environment). Keep secrets in `secrets/`
(mode 0600) and reference them by path.


Archiving
----------------------------------------------------------------

Pick where each backup is shipped and how it is retained with `syncArchive.script`
plus `publishToReady`. The bundled archivers and their parameters:

- [`archive-copy`](archiving-copy.md) — flat, keep the newest N copies.
- [`archive-gfsy`](archiving-gfsy.md) — tiered daily/weekly/yearly on disk.
- [`archive-restic`](archiving-restic.md) — off-box restic repository (dedup,
  encryption).

How synchronous and asynchronous archiving differ, and when to set
`publishToReady` to `false` vs `true`, is covered in [Archiving](archiving.md).


Scheduling
----------------------------------------------------------------

The sequencer plans each window by a fairness score (most overdue first) and packs
what fits. `targetIntervalHours` sets a service's desired interval — set it per
service, or once at the top level as the default (else 24h). When a service has an
archiver, coverage comes from the archiver's own interval instead
(`ARCHIVE_MIN_INTERVAL_HOURS` / `ARCHIVE_DAILY_HOURS` / `RESTIC_MIN_INTERVAL_HOURS`),
so an already-archived service is skipped. A small grace margin (top-level
`coverageMargin`, default 2%) keeps a backup that is *just* under its period from
being skipped by timer jitter. See [the scheduling model](../docs/scheduling.md).


Checking your config
----------------------------------------------------------------

After editing `config.json`, dry-check it before the timer runs:

```sh
bseq validate
```

It prints `OK` or `INVALID` per enabled service, lists each problem (missing
recipe/archiver script, an unregistered step name, an archiver with no `archive`
step, and recipe prerequisites — a restic password file that does not exist, an
unset or unreachable `ARCHIVE_DEST`, a missing compose/data dir), and exits non-zero
if anything is wrong. It runs nothing.

A run applies the same preflight automatically: a service that fails validation is
skipped **before any step runs** — so a broken archive never stops the service for a
backup it cannot keep — and reported as a failure, so coverage leaves it overdue and
the next window retries it once the config is fixed. Run `validate` after any config
change to catch a typo before the nightly timer does.


Resources and timeouts
----------------------------------------------------------------

`resources` (a step holding named locks like `diskA`, `net-out`, declared in the
top-level `resources` map) and `timeoutSec` are **accepted but not yet enforced** —
the executor currently runs steps strictly sequentially, and no step is timed out.
They are forward-looking declarations for a future parallel executor and timeout
support; set them to describe intent, but do not rely on them to gate concurrency
or kill a hung step today. See [the resource model](../docs/architecture.md).
