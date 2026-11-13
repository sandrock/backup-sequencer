systemd units
================================

Sample units to run the sequencer nightly on Linux. The `.timer` starts the
oneshot `.service` at the window open; the run enforces the configured window and
exits (the sequencer is not a daemon). The single-instance lock makes a
double-trigger a harmless no-op.

Before installing, review both files and adjust: the `pwsh` path and code path in
`backup-sequencer.service`, the `User=`, and the `OnCalendar=` schedule in
`backup-sequencer.timer`.


Install
----------------------------------------------------------------

```sh
sudo cp backup-sequencer.service /etc/systemd/system/
sudo cp backup-sequencer.timer   /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now backup-sequencer.timer
```

Check the schedule and run a one-off now to validate the setup:

```sh
systemctl list-timers backup-sequencer.timer      # next firing time
sudo systemctl start backup-sequencer.service     # run once, now (respects the window)
journalctl -u backup-sequencer.service -e         # output of the last run
```

To run a backup now ignoring the window, invoke the sequencer directly with
`-Force` rather than the unit — see the [getting started guide](../../help/getting-started.md).


Quiet logging
----------------------------------------------------------------

Under systemd, PowerShell otherwise floods the journal with its own startup
(`Perftrack_ConsoleStartup`) and script-block-compile lines — separate from the
sequencer's own output. The service's `ExecStart` passes
`-SettingsFile powershell.config.json` (shipped here, `LogLevel: Error`) to silence
them for this session only, leaving the host's global pwsh config untouched. If you
install the code somewhere other than `/opt/backup-sequencer`, fix that path in the
unit.
