Running by hand
================================

`bseq run` from a terminal: when you need options the systemd units don't pass
(`-ForceAllJobs`), or want to watch the output live. For the usual cases,
[the systemd units](systemd.md) are safer: nothing you do in the terminal can
touch the run.


Why not a plain terminal
----------------------------------------------------------------

A run stops services (cold backups) and starts them again at the end of their
backup. Anything that kills or freezes the run in between leaves a service down:

| What happens | Effect on the run |
|---|---|
| Ctrl-C | killed in the middle of a step |
| Ctrl-Z | suspended: everything stops, a stopped service stays down until `fg` |
| SSH connection lost, laptop asleep, terminal closed | killed (the terminal's hangup reaches the run) |

The sequencer does not catch any of these yet. So: always inside *tmux* or *screen*.
They keep the run going on the server whatever happens to your connection; you
come back to it later.

If it happened anyway: [Stopping a run](systemd.md#stopping-a-run) says how to
recover.


tmux
----------------------------------------------------------------

```sh
# New session named bseq, start the run in it
tmux new -s bseq
sudo bseq run -ForceWindow

# Leave it running and get your shell back: Ctrl-b, then d (detach)

# Later, from any login: back to it
tmux attach -t bseq

# Sessions alive on this machine
tmux ls
```

| Keys | Action |
|---|---|
| Ctrl-b d | detach: the session keeps running |
| Ctrl-b [ | scroll mode (arrows, PgUp/PgDn); q to leave |
| Ctrl-b c | new window in the session |
| Ctrl-b n / p | next / previous window |
| `exit` | close the window (ends the session with the last one) |

Detach rather than close the terminal window: closing works too (the session
survives), but detaching is the habit that never surprises.

When the run has finished, `exit` closes the session.


screen
----------------------------------------------------------------

The older equivalent, installed on some hosts where tmux is not.

```sh
screen -S bseq                 # new session named bseq
sudo bseq run -ForceWindow

# Detach: Ctrl-a, then d
screen -r bseq                 # back to it
screen -ls                     # sessions alive
```

Ctrl-a [ scrolls (Esc to leave); `exit` closes the session.


Run options
----------------------------------------------------------------

```sh
sudo bseq run                              # same as the timer: respects the window
sudo bseq run -ForceWindow                 # now, whatever the time
sudo bseq run -ForceWindow -ForceAllJobs   # now, every service, even those recently backed up
```

Follow it from another terminal with `sudo bseq status`; the run's output stays
in the tmux window (not in the journal, unlike a systemd run). The event log
records it like any run: `bseq show latest` afterwards.
