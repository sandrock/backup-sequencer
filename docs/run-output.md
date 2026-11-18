Run output
================================

Why `bseq run` prints what it prints, the way it prints it. The user-facing
description (columns, events) is in `help/following-runs.md`; this page records the
choices behind it, so later changes keep to them or change them on purpose.

Status: implemented (`Format-RunLine`, `Write-RunLine`, `Invoke-BackupStep`,
`Invoke-BackupRun` in `src/lib/BackupCommon.psm1`).


Who reads it
----------------------------------------------------------------

- The admin at a terminal during a manual run, watching what happens now.
- The admin the next morning in the journal (`journalctl -u backup-sequencer`),
  reconstructing a night: what ran, when it started, how long it took, what failed.
- Tomorrow: the same reader, but with several services running in parallel.

The event log (`events.jsonl`) is the record; the run output is the narration of
it, written for people. Nothing parses the run output.


Requirements
----------------------------------------------------------------

1. See each step's start and end, with its status, start time and duration.
2. Dates and times available, without a full timestamp cluttering every line.
3. Parallel-ready: a step may start while others run, so every line must say on
   its own which service and step it is about.
4. Alignment helps the eye: fixed column widths, truncation when too long.
5. No indentation for hierarchy: it doesn't survive a log file, and it can't
   express parallel work (two services "inside" the run at the same time).


Choices
----------------------------------------------------------------

### One line per event, fixed columns

```
runid(4)  [time(8)]  service(18)  step(12)  event(6)  dur(6)  detail
```

Every line is self-contained (requirements 3, 5): the hierarchy run → service →
step is carried by columns, not by position or indentation. Run-level lines use
service `run`; service-level lines use step `-`.

Widths are fixed for the whole program, not computed from the configured names:
the same column sits at the same offset in every run, which also keeps grep and
`cut` usable. Longer values are cut to width with `~` as the last character
(requirement 4). The service name is the one most likely to be cut; it is rarely
ambiguous once cut, and `show` has the full name.

The detail column is free text and never truncated: it carries error messages.

### Run id first

The last 4 characters of the run id open every line. Runs don't overlap (the
single-instance lock), so the id is mostly redundant within one run, but it lets
one night be grepped out of a week of journal (`journalctl ... | grep '^c583'`)
and ties the narration to `bseq show c583`. First, because it is the outermost
context. `----` marks a line written without a run (the "already running" refusal).

### Time: on a terminal only

journald stamps every line with date, time, host and pid, so a time column there
is noise. bseq detects the journal by `$JOURNAL_STREAM` (set by systemd when a
service's stdout is the journal) and drops its own time column and date lines.

On a terminal, each line carries `HH:mm:ss`, and a date line
(`2026-11-18 ------…`) is printed at the start and whenever the date changes —
the backup window crosses midnight (requirement 2). A full ISO timestamp on every
line was rejected: 25 characters of mostly repeated text.

### End lines carry everything

The end line is the one a reader scans, so it repeats the start time:
`ok  53m28s  from 23:01:34` (`from 11-18 23:58:11` when the start was another
day). With interleaved parallel steps the start line may be far above; the end
line must be readable alone (requirement 1).

Durations use the compact `Format-Duration` (`7.5s`, `53m28s`, `1h41m`), which
fits the 6-character column. Estimates (`begin`, `plan`) use the coarser
one-unit `Format-DurationUnit` (`55m`, `1.5h`): they are guesses, precision
would mislead.

### Event words, uppercase for attention

A small fixed vocabulary (`start`, `ok`, `begin`, `skip`, `retry`, `info`,
`closed`, `end`) in lowercase, and uppercase for what needs a human (`FAIL`,
`WARN`, `SKIP` for an abnormal skip such as a full spool). Uppercase stands out
without colour, so it works in the journal and in mail. Words, not symbols:
`grep FAIL` is obvious, `grep '!'` is not.

### Recipe messages are tagged, tool output is not

A step's PowerShell `Write-Warning` / `Write-Information` / `Write-Host` are
captured by `Invoke-BackupStep` and re-emitted as `WARN` / `info` lines with the
service and step columns. The capture forces those preferences on inside the
step's own scope, so a caller silencing its streams never makes a step's warnings
vanish. Other output objects of a step are dropped, as before.

Warnings are also stored on the step's `step.end` event (`warnings`, at most 20,
500 characters each), so `show` and the email report can list them after the
fact; `info` lines are narration only and are not stored.

Native tool output (docker compose, tar) is NOT captured: it goes straight to
stdout/stderr, untagged, between its step's `start` and `ok` lines. Capturing it
(to a per-step file, shown on failure) was considered and deferred — keep it
simple, see how it reads in practice. What is done: the run sets `NO_COLOR=1`,
`COMPOSE_ANSI=never` and `COMPOSE_PROGRESS=plain` (restored afterwards), so
compose prints one line per state change instead of redrawing a progress block.
Environment variables, not flags: a tool that doesn't know them ignores them, so
an old `docker-compose` v1 does not break.

Known cost: once services run in parallel, untagged tool lines will interleave
without saying whose they are. Revisit then (tag each line, or capture to files).

### ASCII only

No arrows, check marks or box drawing: the output goes to the journal, to mail,
and to terminals of unknown encoding.


Out of scope here
----------------------------------------------------------------

- `show` / `status` / `history` and the email report keep their own formats; they
  read the event log after the fact and are not constrained by interleaving.
- Colour on a terminal: not done; uppercase events carry the emphasis.
