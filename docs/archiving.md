Staging and archiving model
================================

How a completed backup is handed off to be archived. This is the boundary between
the sequencer (which produces backups) and archivers (which ship them somewhere
durable).

Status: target design — not yet implemented. The current code uses a single
`wip/` dir and a `cleanup` step; this document describes where it is going. See
`architecture.md` for the as-built engine.


Principle: producer / consumer
----------------------------------------------------------------

The sequencer is only a producer. It builds backups and publishes them to a known
place. It never ships them anywhere, and the only backup it ever deletes is a
stale, unconsumed one it is replacing at publish time (with a notification) —
every other deletion belongs to the archiver, after a successful ship.

Archiving is done by separate consumers, and is fully pluggable — FTP push, a
remote pull over ssh/rsync/sshfs, restic to a repository, object storage, tape.
The sequencer makes no assumption about the method. This keeps the tool generic
and the sequencer free of archiving concerns.


The staging lifecycle
----------------------------------------------------------------

A backup is a *set of artifacts* (e.g. a compose dir, binaries, data, a database
dump) — not a single file. Keeping them separate lets each be restored on its own
and lets a dedup archiver (restic) skip the ones that didn't change.

The staging root has top-level phase dirs; each holds one subdir per service:

```
<staging-root>/
  wip/      dumps being built              wip/<service>/
  stage/    complete; sync-archive step     stage/<service>/
  ready/    done; awaiting async archiver    ready/<service>/
  trash/    being deleted; reaped async      trash/<service>-<id>/
```

A backup moves `wip/<service> → stage/<service> → ready/<service>` by atomic
directory rename (same filesystem). The service name appears once, as the subdir
under whichever phase the backup currently occupies. `ready/<service>/` holds the
several artifacts of that backup.

The engine owns the promotions:
1. backup steps (e.g. stop / dump / start) write the artifact set into
   `wip/<service>/`;
2. engine: atomic `wip/<service> → stage/<service>` (the set is complete);
3. the optional sync-archive step runs against `stage/<service>/`;
4. engine: atomic `stage/<service> → ready/<service>` (publish).

Because the rename is atomic, a consumer of `ready/` never sees a half-written
set — it sees the complete set or nothing.


Two archiver shapes
----------------------------------------------------------------

Primary — the async archiver (out of band). A separate process, local or remote,
watches `ready/`, takes each `ready/<service>/`, ships it, and deletes it. It runs
on its own schedule; the sequencer does not invoke it. This is the path users are
expected to set up.

Optional — the sync archive step (in run). A step that runs during the backup,
in the `stage` phase, shipping the set synchronously (e.g. an FTP push to an
existing destination). It is a shared, optional step — empty by default, so a
backup flows `wip → stage → ready` with nothing in between. Its purpose is
transitional: let an existing archiving process keep working while a proper async
archiver is not yet in place.

Note the two never collide on one backup: while the sync step runs, the backup is
in `stage/`; it only reaches `ready/` after the sync step completes. The async
archiver only ever looks at `ready/`, so it can never touch a backup that is still
being sync-archived. The phase boundary is the coordination — no lock needed.


The `ready/` contract
----------------------------------------------------------------

- Completeness: anything under `ready/<service>/` is a complete artifact set
  (guaranteed by the atomic directory rename).
- Deletion ownership: the async archiver deletes `ready/<service>/` after a
  successful ship. The sequencer deletes a published backup only in one case —
  replacing a stale, unconsumed one at publish time (overwrite-and-notify, below).
- Bounded, never accumulating: `ready/` holds at most the latest backup per
  service. If the previous one is still there when a new one is published (the
  archiver has not taken it), the engine replaces it and emits a notification
  ("archiver has not kept up") rather than piling copies up. One local copy only.
- One physical copy end to end: the backup is moved between phases by rename, not
  copied — safe for large backups.


Deletion is a rename, reaped asynchronously
----------------------------------------------------------------

Removing a backup set can be slow (many gigabytes, many inodes), and some of those
removals happen on the backup critical path — the engine's replace-and-publish
runs inside the time window. So "delete" is not an inline `rm`; it is an atomic
rename into `trash/` (instant), and a reaper empties `trash/` opportunistically,
off the hot path (e.g. at the start of a run, or as a low-priority background
task), where it does not compete with publishing.

Both deleters use it: the engine's replace-delete and, if it wishes, the
archiver's post-ship delete both just rename into `trash/`. This keeps every
delete O(1) on the path that matters and lets the actual freeing happen when there
is slack. The reaper is the engine's one piece of deletion work — and it only ever
removes things already moved to `trash/`, never a live `ready/` set.


The replace-while-fetching race
----------------------------------------------------------------

Publishing is atomic, but replacing an existing `ready/<service>/` while a
consumer is reading it is the one case to handle deliberately (a directory, unlike
a single file, cannot be swapped out from under a reader for free). Two
resolutions; pick by deployment:

1. Schedule apart (simple, dumb-consumer friendly). Run the async archiver outside
   the backup window. Producer and consumer never touch `ready/<service>/` at the
   same time, so the producer can replace safely (rename old into `trash/`, publish
   new) and the consumer can be a plain `scp -r` / `rsync` + delete. No markers, no
   claim.
2. Claim by rename (overlap-safe). The consumer first does one atomic
   `mv ready/<service> <its-workspace>/` to claim the set, then ships at leisure,
   then deletes. The producer only ever touches *unclaimed* dirs in `ready/`, so it
   can never disturb a set being read. Costs the archiver one rename; not a pure
   blind copy.

The producer side is identical either way; only the archiver's behaviour differs.


Transports for an off-host archiver
----------------------------------------------------------------

For a dedup archiver (restic) shipping to another machine, the transport affects
efficiency — a plain copy re-sends the whole artifact every run, while a dedup
archiver can store (and sometimes transfer) only what changed:

- Dedup on the source: run the dedup tool on the producer host, writing to a
  repository exposed from the destination (e.g. restic to a rest-server reached
  over a reverse SSH tunnel). Chunks locally, transfers only new blocks — most
  network-efficient; the producer holds the repository key.
- Dedup on the archiver, reading the source over a mount: the archiver mounts the
  producer's `ready/` (e.g. sshfs) and runs the dedup tool against it, writing to a
  repository local to the archiver. Keeps the producer dumb; the repository and key
  live on the archiver. The source is read in full over the mount each run, so
  network read is not reduced — only repository storage is deduped.
- An `sftp`-style backend is a middle option (the tool drives ssh itself, no fuse
  mount).

The sequencer is indifferent to which is used — it only guarantees the `ready/`
contract above.


Engine vs recipe responsibilities
----------------------------------------------------------------

- Engine: the atomic promotions, the `ready/` contract (bound, replace + notify),
  running the optional sync-archive step in the `stage` phase, and reaping
  `trash/`.
- Recipe: produce the artifact set into `wip/<service>/` (stop / dump / start).
  Recipes do not deal with staging phases or archiving.
- Sync-archive step: a shared, optional step-script (not duplicated per recipe),
  selected per service in config, run in the `stage` phase.
- Async archiver: entirely outside the sequencer.


Concurrency of runs
----------------------------------------------------------------

Two sequencer runs never overlap: the single-instance lock (a PID lock file,
reclaimed if stale) makes a second run exit immediately. So the staging dirs are
only ever advanced by one producer at a time.
