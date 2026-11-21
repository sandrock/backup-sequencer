Running under systemd
================================

Day-to-day control of the sequencer through its systemd units: starting a run,
following it, the timer, the journal. Installing the units:
[`packages/systemd/`](../packages/systemd/README.md).

Three units:

- `backup-sequencer.timer` fires once a night at the window open;
- `backup-sequencer.service` is one run: `bseq run`, then it exits;
- `backup-sequencer-now.service` is one run outside the window:
  `bseq run -ForceWindow`. Started by hand only, no timer.

To run `bseq` from a terminal instead, see [Running by hand](manual-runs.md).


Starting a run now
----------------------------------------------------------------

```sh
# Tonight's plan, now, within the window: outside it, the run exits with nothing to do
sudo systemctl start --no-block backup-sequencer.service

# A run now, whatever the time (bseq run -ForceWindow)
sudo systemctl start --no-block backup-sequencer-now.service
```

Why `--no-block`: the services are `Type=oneshot`. For a daemon, "started"
means the process is up, so `systemctl start` returns at once. A oneshot is
"started" only once its command has finished, so without `--no-block`, `start`
waits for the whole run (minutes to hours; the unit shows `activating`
meanwhile). Either way the run belongs to systemd: closing the terminal or the
SSH session does not stop it.

A start while a run is in progress (from either unit, the timer or a terminal) is
a no-op: the second run finds the lock and exits.

`backup-sequencer-now.service` only lifts the window. Coverage and the fit check
still apply: a service backed up recently is skipped. To back up everything
regardless (`-ForceAllJobs`), run by hand.


Watching it
----------------------------------------------------------------

```sh
sudo bseq status                                  # service, step, elapsed (best view)
systemctl status backup-sequencer.service         # activating = running; last lines of output
                                                  # (backup-sequencer-now.service for a run started with it)
journalctl -u 'backup-sequencer*' -f              # follow the output live (Ctrl-C stops following, not the run)
```


Past runs in the journal
----------------------------------------------------------------

```sh
journalctl -u backup-sequencer -e                         # the end: the last nightly run
journalctl -u backup-sequencer-now -e                     # the last run started with -now
journalctl -u backup-sequencer --since today
journalctl -u backup-sequencer --since "2026-01-14 23:00" --until "2026-01-15 08:00"
journalctl -u backup-sequencer -o cat --since today       # without the journal's prefix
journalctl -u backup-sequencer --since today | grep 'c583'   # one run, by the start of its id
```

`-u 'backup-sequencer*'` takes both services at once. The journal keeps what the
run printed. The event log (`bseq show`, `bseq
history`) is the record that lasts; the journal may be rotated away.


The timer
----------------------------------------------------------------

```sh
systemctl list-timers backup-sequencer.timer      # LAST and NEXT firing
systemctl is-enabled  backup-sequencer.timer       # enabled = fires every night

sudo systemctl disable --now backup-sequencer.timer   # pause the nightly runs
sudo systemctl enable  --now backup-sequencer.timer    # resume them
```

`Persistent=true` in the sample timer: if the machine was off, or the timer
disabled, at the firing time, enabling it again (or booting) fires a catch-up
run at once. That run still respects the window.

Changing the schedule (it must match the window open in the config):

```sh
sudo systemctl edit backup-sequencer.timer
```

```ini
[Timer]
OnCalendar=
OnCalendar=*-*-* 19:00
```

The empty `OnCalendar=` clears the shipped value; without it both times fire.
`systemctl edit` reloads systemd itself; check with `list-timers`.


Stopping a run
----------------------------------------------------------------

Avoid it. `sudo systemctl stop backup-sequencer.service` (or `-now`) kills the run and
everything it started (tar, `docker compose`, restic) in the middle of a step.
The sequencer does not catch it: a service stopped for a cold backup stays down,
and the run shows as interrupted.

If you had to:

```sh
sudo bseq status        # "run ... was interrupted": which service and step
sudo bseq show <run>    # what each service had done
```

Then start by hand what the run had stopped (for docker-compose-cold:
`docker compose up -d` in its compose dir). The next run cleans up the spool's
leftovers.
