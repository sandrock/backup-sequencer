Following runs
================================

Three commands tell you what the sequencer is doing and what it did. None of them
changes anything; they read the event log (`events.jsonl` in the state dir) and the
lock file.

```sh
bseq status                 # the run in progress, or the last run when idle
bseq show <run>...          # per-service detail of one or more runs
bseq history                # recent runs, newest first (-Limit <n>, default 20)
```


Live run output
----------------------------------------------------------------

`bseq run` prints one line per event, in fixed columns, with no indentation: every
line names its run, service and step, so it stays readable in the journal and when
steps of different services interleave.

```
2026-11-18 ----------------------------------------------------------------------
c583  23:01:09  run                 -             begin                 20261118-230108-c583  window 23:00-06:00
c583  23:01:09  photos              -             skip                  covered (x0.70)
c583  23:01:10  wiki                -             begin                 est 55m
c583  23:01:10  wiki                stop          begin
 Container wiki-app-1  Stopping
 Container wiki-app-1  Stopped
c583  23:01:34  wiki                stop          end     ok    24s     from 23:01:10
c583  23:01:34  wiki                dump          begin
c583  23:55:02  wiki                dump          end     ok    53m28s  from 23:01:34
c583  23:58:11  wiki                hot-dump      msg     WARN          wiki hot-dump: some files changed while being read
2026-11-19 ----------------------------------------------------------------------
c583  00:41:52  wiki                -             end     ok    1h41m   51.5 GB  snap a5090c0f, local copy dropped
c583  00:41:53  chat                -             begin                 est 4m
c583  00:41:53  chat                dump          begin
c583  00:42:20  chat                dump          end     FAIL  27s     from 00:41:53  ! pg_dump exited with code 1
c583  00:42:20  chat                start         begin
c583  00:42:31  chat                start         end     ok    11s     from 00:42:20
c583  00:42:31  chat                -             end     FAIL  38s     ! step dump failed; partial dump discarded
c583  00:43:10  run                 -             end     FAIL  1h42m   1 ok, 1 failed, 7 planned
```

Columns: the last 4 characters of the run id, the time, the service (`run` for the
run itself), the step (`-` for a service- or run-level line, `[trash]` / `[delete]`
for the engine's own deletions, see below), the event, its
status, the duration, then free text. Names longer than the column are cut with
`~`. An error message always starts with `!`; on a step's `end` line it follows
the step's start time.

| Event | Meaning |
|---|---|
| `begin` / `end` | a step, a service or the run begins / ends; a service's `begin` gives its estimated duration (`?` when never backed up), an `end` line repeats the start time (`from ...`) |
| `skip` | a service is not run: `covered` (already backed up for its period), `deferred` (won't fit the time left), or out of spool space |
| `retry` | a failed step is tried again |
| `msg` | a message from the recipe during that step, or from the engine about the service |
| `closed` | the window closed: remaining services wait for the next run |
| `done` | an engine action with no begin line: `[trash]`, `[delete]`, or `[notify]` (the report mail, after the run's `end`) |

| Status | Meaning |
|---|---|
| `ok` | the step, service or run succeeded |
| `FAIL` | it failed; the reason follows `!` |
| `WARN` | needs attention (a recipe warning, a full spool, another run already in progress) |
| (blank) | nothing to report |

Uppercase statuses need attention: `grep -E 'FAIL|WARN'` finds them all. A run ends
`FAIL` when any service failed. Output from the tools a step calls (docker
compose, tar) passes through as is, between that step's `begin` and `end` lines;
bseq asks them for plain output (no colours, no redrawn progress).

### Deletions

Nothing is deleted silently. A backup dir that is no longer needed is first moved
into the spool's `trash/` (instant), and deleted at the start of the next run.
For the archived local copy, `emptyTrash` can bring that forward to the end of the
service or of the run, or leave it to you (see
[Configuring a service](configuring-services.md#freeing-the-spool-emptytrash)):

```
c583  run                 -             begin                 20261118-230108-c583  window 19:00-06:00
c583  forgejo             [delete]      done    ok    12s     47.0 GB, 12345 files  local-copy-dropped, run a558
c583  run                 [delete]      done    ok    1.0s    120.4 MB, 80 files  no record: wip-3f2a9c1b77e0
c583  bookstack           [delete]      done    FAIL  0.1s    ! bookstack-1a2b3c4d5e6f: Access to the path is denied; kept, retried next run
c583  grafana             [trash]       done    ok            wip leftover (interrupted run?) -> trash/wip-9c0e11aa2b3c
c583  keycloak            sync-archive  msg                   pruned /srv/archive/keycloak/20261104-230012-77aa (1.2 GB, 34 files)
c583  keycloak            sync-archive  msg     WARN          prune failed, kept for next run: /srv/archive/keycloak/20261103-230009-12cd: Directory not empty
c583  run                 -             end     ok    39m58s  5 ok, 0 failed, 5 planned  freed 47.1 GB (2 items), 1 trash item kept
```

- `[delete]`: an item of `trash/` deleted, with its size, why it was there and
  which run put it there (`no record` for items older than this logging).
  A failed delete leaves the item in `trash/`; the next run tries again.
- `[trash]`: a dir moved into `trash/`, shown only when unusual: a leftover from
  an interrupted run, the partial dump of a failed backup, a backup in `ready/`
  replaced before the archiver took it, a backup whose archive failed (the only
  copy left), cleanup after an error. The routine drop of the local copy once
  archived is recorded in the event log, not printed.
- With `emptyTrash` set to `service-end` or `run-end`, the `[delete]` line of the
  archived copy comes in the same run, right after the service's `end` line or
  just before the run's.
- With `emptyTrash: manual`, nothing is deleted; the archived copy gets a line
  instead, and the run's end line counts what the trash holds for you:

  ```
  c583  mattermost          [delete]      skip                  held (emptyTrash manual): 31.2 GB, 2 files; bseq trash -Delete mattermost-5d1e0c9a2b47
  c583  run                 -             end     ok    1h02m   4 ok, 0 failed, 4 planned  freed 1.1 GB (1 item), 2 held in trash (62.4 GB)
  ```
- Archive prunes (`archive-copy`, `archive-gfsy`) print what they delete, with its
  size. A prune that fails is a warning: the service stays `ok`, the old archive
  stays recorded as live, and the next run prunes it again.
- A lock left by a run that died is removed with a `WARN` line naming its pid.

To see what the trash holds, and empty it without waiting for a run:

```sh
bseq trash                              # list: size, service, why, which run, when
bseq trash -Delete                      # delete everything now (as root)
bseq trash -Delete forgejo-0123456789ab # only the items named
```

```
trash  /var/spool/backup-sequencer/trash   2 items, 32.9 MB
  host-config-5468a97d02ac        30.7 MB      24 files  next run  host-config  wip-leftover         run caa1  moved 2026-11-18T15:14:14+01:00
  junk                               0 KB       0 files  next run  no record
last deletions:
  2026-11-18T15:14:12+01:00  run caa1      2.2 MB  vaultwarden-b4056de97dcb
```

The column after the file count says when a run deletes the item: `next run`, or
`manual` for a service set to `emptyTrash: manual` (only `-Delete` frees it).

`-Delete` takes the run lock: it is refused while a run is in progress, and a run
starting meanwhile is refused like a second run would be. Its deletions are
logged like a run's, marked `by command`. Listing is read-only and usually works
without `sudo`, like `show`.

Under systemd the output goes to the journal, which stamps each line itself, so
bseq drops its own time column and date lines there:

```sh
journalctl -u backup-sequencer --since today
```


bseq status
----------------------------------------------------------------

During a run, `status` shows which service and step is working and for how long:

```
RUNNING  run 20261117-152140-332d  since 2026-11-17T15:21:40+00:00 (14m05s)  pid 81234
  vaultwarden            ok                      7.2 MB  snap 5ef12f46  stop 7.5s, dump 3.1s, start 4.2s, sync-archive 2.0s
  baget                  ok                    269.6 MB  snap e237422b  stop 4.0s, dump 12s, start 7.0s, sync-archive 20s
  teamcity               > hot-dump (4m10s)                             stop 49s, dump 5m02s, start 31s
```

`>` marks the step running now, with the time spent in it. Services the run has
not reached yet show as `pending`.

When nothing runs, it shows the last run, prefixed `idle  last run …`, and mentions
any earlier run that was interrupted.

### An interrupted run

A run that started but never recorded its end, while no process holds the lock,
was interrupted — the machine rebooted, the process was killed, or it crashed:

```
INTERRUPTED  run 20261117-230001-1a2b  started 2026-11-17T23:00:01+00:00; no end recorded and no process holds the lock
  teamcity               ! died in dump
      ! check this service: a backup stopped mid-step may have left it down
  nextcloud              not reached
```

Act on the `! died in …` line: if the run stopped between `stop` and `start`, the
service may still be down. For a docker-compose service:

```sh
cd <compose dir> && docker compose ps      # down? then:
docker compose up -d
```

Nothing else needs cleaning up. The next backup of that service discards the
half-finished working dirs it left in the spool, and a service without a successful
backup stays due, so it is retried automatically.


bseq show
----------------------------------------------------------------

The same per-service view for any run, finished or not:

```sh
bseq show                    # the latest run
bseq show latest
bseq show 20261117-152140-332d
bseq show 332d               # any unique ending of the run id
bseq show a48a 6e97          # several runs, one after the other
```

An ending that matches several runs is refused and lists them; type a few more
characters. Run ids are shown by `bseq history`.


bseq history
----------------------------------------------------------------

One line per run, newest first, with its id so you can `bseq show` it:

```
2026-11-17T23:00:01+00:00  20261117-230001-1a2b  INTERRUPTED  [teamcity:dump...]
2026-11-17T15:58:12+00:00  20261117-152140-332d  ok=3 failed=0 planned=3  [vaultwarden:ok baget:ok teamcity:ok]
2026-11-16T23:01:23+00:00  20261116-230000-9f3e  ok=0 failed=0 planned=1  [vaultwarden:covered]
```

- A finished run shows its end time and counts.
- A running or interrupted run shows its start time; `svc:step...` is the step it
  is in (or died in).
- In the brackets, each service shows its outcome, or why it was skipped.


Reading a service line
----------------------------------------------------------------

```
  baget                  ok                    269.6 MB  snap e237422b  stop 4.0s, dump 12s, start 7.0s, sync-archive 20s
```

From left to right: the service, its outcome, the size of the backup set, the
archive id, and the steps in run order with their durations.

Outcome:

| Shown | Meaning |
|---|---|
| `ok` | backed up (and archived, when an archiver is configured) |
| `FAIL` | a step or the archive failed; the reason is on the lines below |
| `INVALID` | refused by the preflight check before any step ran — no downtime; the problems are listed below |
| `skipped (covered)` | already backed up recently enough; nothing to do |
| `skipped (deferred)` | would not fit in what was left of the window; runs in a later window |
| `skipped (insufficient-space)` | not enough free space in the spool for the expected size |
| `> step (time)` | running now |
| `! died in step` | the run was interrupted during that step (see above) |
| `> between steps` / `! died between steps` | the service has begun but no step is running: the engine is between two steps or archiving / publishing; or the run died there |
| `pending` / `not reached` | not started yet / never started (the run ended or died first) |

The archive column is the restic snapshot id (`snap …`), or the tier for tiered
on-disk archives. A failed step shows as `FAIL` next to its duration
(`sync-archive FAIL 1s`). Lines starting with `!` under a service give the error,
and the warnings of that service in the run: the recipe's own, prefixed with their
step, and the engine's (a backup replaced in `ready/`, a spool out of space):

```
  forgejo                ok                    1.2 GB    snap 41c09e7a  stop 3.0s, dump 2m10s, start 5.1s
      ! dump: tar: ./data/queues/000123.log: file changed as we read it
```

The email report lists the same warnings, one line each. A step keeps at most 20
of them in the event log, each cut to 500 characters.

What the run deleted comes last, in one block; a `moved to trash:` line under a
service lists its unusual moves (see Deletions above):

```
  grafana                FAIL                                  dump FAIL 1m46s
      ! step dump failed; partial dump discarded
      moved to trash: failed-dump
  deletions              freed 47.1 GB in 2 items, pruned 1 archive (1.2 GB); 2 failed
      delete  forgejo            47.0 GB   12345 files  local-copy-dropped, run a558
      delete  (no record)       120.4 MB      80 files  no record: wip-3f2a9c1b77e0
    ! delete  bookstack        kept: bookstack-1a2b3c4d5e6f: Access to the path is denied
      prune   keycloak            1.2 GB      34 files  /srv/archive/keycloak/20261104-230012-77aa
    ! prune   keycloak         kept: /srv/archive/keycloak/20261103-230009-12cd: Directory not empty
```

The email gives the totals line and one `!` line per item it could not delete.


Permissions
----------------------------------------------------------------

`status`, `show` and `history` only read the state dir. On an installed system they
usually work without `sudo` (the event log is world-readable); `run` needs root.
