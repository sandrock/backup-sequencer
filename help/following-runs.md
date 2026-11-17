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
| `pending` / `not reached` | not started yet / never started (the run ended or died first) |

The archive column is the restic snapshot id (`snap …`), or the tier for tiered
on-disk archives. A failed step shows as `FAIL` next to its duration
(`sync-archive FAIL 1s`). Lines starting with `!` under a service give the error.


Permissions
----------------------------------------------------------------

`status`, `show` and `history` only read the state dir. On an installed system they
usually work without `sudo` (the event log is world-readable); `run` needs root.
