dir-tar recipe
================================

A tar of one or more directories, taken while the service runs, in the order you
list them. Bundled at `recipes/dir-tar.ps1`. Nothing is stopped: no downtime.

Use it for data that tolerates a live copy: write-once stores, file shares, a
container registry. For data that must be quiet while copied (a database), stop
the service first (see [docker-compose-cold](recipe-docker-compose-cold.md)).


What it does
----------------------------------------------------------------

One step, `dump`, which writes `<service>.tar` (uncompressed: compression and
dedup happen in the archiver).

- Every `DT_DIRS` entry goes in, **in the order given**. When one part of a store
  references another, list the referencing part first; see the registry example.
- Paths are stored relative to `/` (`srv/registry/...`), so the tar says where its
  files go back. Owners are stored as numbers (`--numeric-owner`).
- A file that changes or disappears while tar reads it gives a warning, not a
  failure: it is a live copy. Any other tar error fails the step.


Parameters
----------------------------------------------------------------

| Variable | Required | Default | Meaning |
|---|---|---|---|
| `DT_DIRS` | yes | — | absolute dirs, `;`-separated, archived in this order |
| `DT_EXCLUDES` | no | — | `;`-separated. An entry starting with `/` is an absolute path (left out exactly); any other entry is a name pattern matched anywhere (`*.tmp`, `cache`) |
| `DT_ARCHIVE_NAME` | no | `<service>.tar` | the tar's name |

`bseq validate` (and every run, before the step) flags a `DT_DIRS` entry that is
not absolute, missing, not a directory, `/` itself, or inside another entry
(it would be archived twice). The service is then skipped, not backed up partly.


Example: a container registry
----------------------------------------------------------------

A Docker registry (distribution) on disk keeps layers in `v2/blobs` and, in
`v2/repositories`, small link files saying which tag uses which blob. A push
writes blobs before the links that point to them. Tar `repositories` first:
every link in the tar then has its blob, even with pushes during the copy.

```json
"registry-images": {
  "enabled": true,
  "targetIntervalHours": 168,
  "script": "recipes/dir-tar.ps1",
  "env": {
    "DT_DIRS": "/srv/zot/docker/registry/v2/repositories;/srv/zot/docker/registry/v2/blobs",
    "DT_EXCLUDES": "_uploads"
  },
  "publishToReady": false,
  "syncArchive": { "script": "recipes/archive-restic.ps1" },
  "steps": [
    { "name": "dump", "resources": ["diskA"] }
  ]
}
```

`_uploads` holds pushes in progress: not worth keeping. Garbage collection
deletes blobs; do not run it during the backup.


Restore
----------------------------------------------------------------

```sh
# Look first
tar -tvf registry-images.tar | head

# Into a scratch dir, to check or pick files
mkdir -p /tmp/restore && tar --numeric-owner -xpf registry-images.tar -C /tmp/restore

# In place: stop what uses the dirs, move them aside, then
tar --numeric-owner -xpf registry-images.tar -C /
```
