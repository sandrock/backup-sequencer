docker-compose-cold recipe
================================

Cold backup of a docker-compose service: stop the stack, archive it, start it
again. Bundled with the sequencer at `recipes/docker-compose-cold.ps1`.


What it does
----------------------------------------------------------------

Three steps, run while the service is down:

1. `stop` — if the stack is running, `docker compose down`. If it is already down,
   it is left alone (and `start` will not bring it up).
2. `dump` — write the backup set as two uncompressed tars (below).
3. `start` — bring the stack back up **only if `stop` took it down**. A service
   that was already down at backup time stays down.

List the steps you want in the service's `steps` array. Give `start`
`"alwaysRun": true` so the stack comes back even if `dump` failed.


What it archives
----------------------------------------------------------------

The `dump` step produces two artifacts, keeping configuration and data apart so a
service can be restored on its own:

- `<service>_compose.tar` — the compose directory (`docker-compose.yml`, `.env`,
  and siblings). Any data dir that lives *inside* the compose dir is excluded
  automatically (it is already in the data tar), as is anything listed in
  `DCC_COMPOSE_EXCLUDES` (e.g. `logs`).
- `<service>_data.tar` — every directory in `DCC_DATA_DIRS`, each stored under its
  own leaf name. The dirs may live on different volumes and still share one tar —
  this covers data kept off the system disk for space.

Both tars are uncompressed; compression, if any, is left to the archiver/transport
downstream.


Configuration
----------------------------------------------------------------

Set these in the service's `env`:

| Env | Required | Default | Purpose |
|---|---|---|---|
| `DCC_COMPOSE_DIR` | yes | — | the compose dir (`docker-compose.yml` + siblings) |
| `DCC_COMPOSE_EXCLUDES` | no | — | extra paths (relative to the compose dir) to keep out of the compose tar; `;`-separated |
| `DCC_COMPOSE_CMD` | no | `docker compose` | the compose command |
| `DCC_COMPOSE_ARCHIVE_NAME` | no | `<service>_compose.tar` | compose tar file name |
| `DCC_DATA_DIRS` | yes | — | data dir(s) to archive; `;`-separated, may span volumes |
| `DCC_DATA_ARCHIVE_NAME` | no | `<service>_data.tar` | data tar file name |

The legacy single-dir `DCC_DATA_DIR` and the old `DCC_ARCHIVE_NAME` are still
accepted as fallbacks.

Example:

```json
"vaultwarden": {
  "enabled": true,
  "mandatory": true,
  "script": "recipes/docker-compose-cold.ps1",
  "env": {
    "DCC_COMPOSE_DIR": "/srv/vaultwarden",
    "DCC_DATA_DIRS": "/srv/vaultwarden/data;/mnt/vol2/vaultwarden-ssl",
    "DCC_COMPOSE_EXCLUDES": "logs"
  },
  "steps": [
    { "name": "stop",  "resources": [] },
    { "name": "dump",  "resources": ["diskA"], "timeoutSec": 1800 },
    { "name": "start", "resources": [], "alwaysRun": true }
  ]
}
```

Decide deliberately where TLS certs and similar go: list a dir in `DCC_DATA_DIRS`
and it rides in the data tar; leave it under the compose dir (and out of
`DCC_DATA_DIRS`) and it rides in the compose tar.


Restoring
----------------------------------------------------------------

There is no restore command yet — restore by extracting each artifact back to
where it came from, then starting the stack.

The compose tar holds the compose dir by its leaf name, so extract it into the
compose dir's parent:

```sh
tar -C /srv -xf vaultwarden_compose.tar          # recreates /srv/vaultwarden/…
```

The data tar holds each data dir by its leaf name. If they shared a parent,
extract there; if they came from different volumes, extract to a scratch dir and
move each leaf back to its parent:

```sh
tar -C /tmp/restore -xf vaultwarden_data.tar
mv /tmp/restore/data           /srv/vaultwarden/data
mv /tmp/restore/vaultwarden-ssl /mnt/vol2/vaultwarden-ssl
```

Then bring it back up:

```sh
cd /srv/vaultwarden && docker compose up -d
```

(A manifest recording the exact create/restore command per archive is planned, so
this becomes copy-paste.)
