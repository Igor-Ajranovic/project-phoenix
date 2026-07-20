# systemd Service and Timer Lab

## Document Information

| Field | Value |
|---|---|
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Hypervisor | Proxmox VE |
| Main topics | systemd services, timers, shell scripts |
| Status | Completed and verified |

## Purpose

This lab demonstrates how systemd can execute a custom administrative script:

- manually;
- during system boot;
- on a recurring schedule;
- with centralized logging and status inspection.

A simple heartbeat script was used as a safe example.

The heartbeat itself is not the final objective. The real objective is to understand the relationship between:

```text
script
service unit
timer unit
systemd
journal
```

## Architecture

The final execution chain is:

```text
phoenix-heartbeat.timer
        ↓
phoenix-heartbeat.service
        ↓
/usr/local/bin/phoenix-heartbeat.sh
        ↓
/var/log/phoenix-heartbeat.log
```

The timer controls when the task runs.

The service defines how the task runs.

The script performs the actual work.

The log file stores the result.

## Custom Script

The script was created at:

```text
/usr/local/bin/phoenix-heartbeat.sh
```

Contents:

```bash
#!/usr/bin/env bash

echo "$(date --iso-8601=seconds) phoenix-linux-01 heartbeat" >> /var/log/phoenix-heartbeat.log
```

The script appends:

- an ISO 8601 timestamp;
- the system hostname;
- a heartbeat message.

Example output:

```text
2026-07-20T21:20:39+00:00 phoenix-linux-01 heartbeat
```

## Why `/usr/local/bin`

The `/usr/local/bin` directory is appropriate for locally created administrative scripts.

It separates custom tools from files managed by the operating system package manager.

Typical use cases include:

- backup scripts;
- health-check scripts;
- maintenance commands;
- monitoring helpers;
- locally developed automation.

The script could technically be stored elsewhere, but the systemd `ExecStart` directive must always reference the correct absolute path.

## Script Permissions

The script was made executable with:

```bash
sudo chmod 755 /usr/local/bin/phoenix-heartbeat.sh
```

The permissions were verified with:

```bash
ls -l /usr/local/bin/phoenix-heartbeat.sh
```

Expected permission state:

```text
-rwxr-xr-x
```

The file is owned by `root` and executable by the system.

## Manual Script Test

The script was tested before introducing systemd:

```bash
sudo /usr/local/bin/phoenix-heartbeat.sh
sudo tail -n 1 /var/log/phoenix-heartbeat.log
```

This confirmed that:

- the script executed successfully;
- the log file could be created;
- the timestamp format was valid;
- the expected heartbeat entry was written.

Testing the script independently prevents systemd from hiding a basic script problem.

## Service Unit

The following service unit was created:

```text
/etc/systemd/system/phoenix-heartbeat.service
```

Contents:

```ini
[Unit]
Description=Project Phoenix heartbeat logger
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/phoenix-heartbeat.sh

[Install]
WantedBy=multi-user.target
```

## Service Unit Explanation

### `[Unit]`

```ini
Description=Project Phoenix heartbeat logger
```

Provides a human-readable description.

```ini
After=network.target
```

Places the service after the basic network target in the boot ordering.

The current heartbeat script does not strictly require networking, but this directive was included to demonstrate dependency ordering.

### `[Service]`

```ini
Type=oneshot
```

The service runs one command and exits.

A successful `oneshot` service normally returns to:

```text
inactive (dead)
```

This is expected and does not indicate failure.

```ini
ExecStart=/usr/local/bin/phoenix-heartbeat.sh
```

Defines the command executed by systemd.

### `[Install]`

```ini
WantedBy=multi-user.target
```

Allows the service to be connected to the normal multi-user boot target when enabled.

## Loading the Service

After creating the unit file, systemd configuration was refreshed:

```bash
sudo systemctl daemon-reload
```

The service was inspected before execution:

```bash
systemctl status phoenix-heartbeat.service --no-pager
```

Initial state:

```text
Loaded: loaded
Active: inactive (dead)
```

This confirmed that systemd recognized the unit but had not yet executed it.

## Manual Service Test

The service was started manually:

```bash
sudo systemctl start phoenix-heartbeat.service
```

Its status was inspected:

```bash
systemctl status phoenix-heartbeat.service --no-pager
```

Relevant result:

```text
Deactivated successfully
Finished phoenix-heartbeat.service
```

The log file was then checked:

```bash
sudo tail -n 2 /var/log/phoenix-heartbeat.log
```

This confirmed that systemd successfully executed the script.

## Direct Boot Enablement Test

The service was temporarily enabled:

```bash
sudo systemctl enable phoenix-heartbeat.service
```

Its relationship with the boot target was verified:

```bash
systemctl list-dependencies multi-user.target | grep phoenix-heartbeat
systemctl is-enabled phoenix-heartbeat.service
```

The VM was rebooted and the log was reviewed.

A new heartbeat entry appeared during boot, confirming that the service persisted across restart.

This direct boot mechanism was later disabled because the timer became the preferred automation method.

## Timer Unit

The following timer was created:

```text
/etc/systemd/system/phoenix-heartbeat.timer
```

Contents:

```ini
[Unit]
Description=Run Project Phoenix heartbeat every 5 minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Unit=phoenix-heartbeat.service

[Install]
WantedBy=timers.target
```

## Timer Unit Explanation

### `OnBootSec=2min`

The first scheduled execution occurs approximately two minutes after boot.

### `OnUnitActiveSec=5min`

The timer schedules the next execution five minutes after the service was last activated.

### `Unit=phoenix-heartbeat.service`

The timer activates the existing service unit.

The timer does not run the script directly.

### `WantedBy=timers.target`

Allows the timer to start automatically during the normal system boot process.

## Enabling the Timer

The new timer definition was loaded and enabled:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now phoenix-heartbeat.timer
```

The timer state was checked:

```bash
systemctl status phoenix-heartbeat.timer --no-pager
```

Confirmed state:

```text
Active: active (waiting)
```

This means the timer is running and waiting for its next scheduled activation.

## Timer Schedule Verification

The schedule was inspected with:

```bash
systemctl list-timers phoenix-heartbeat.timer
```

The output displayed:

- the next execution time;
- the remaining wait time;
- the previous execution time;
- the timer unit;
- the service activated by the timer.

The heartbeat log was checked with:

```bash
sudo tail -n 5 /var/log/phoenix-heartbeat.log
```

A new timestamp confirmed that the timer successfully triggered the service.

## Timer Lifecycle Tests

### Stop without disabling

```bash
sudo systemctl stop phoenix-heartbeat.timer
```

Result:

```text
inactive
```

The timer stopped immediately but remained enabled for future boots.

### Check boot enablement

```bash
systemctl is-enabled phoenix-heartbeat.timer
```

Result:

```text
enabled
```

### Start without changing boot configuration

```bash
sudo systemctl start phoenix-heartbeat.timer
```

Result:

```text
active
```

### Disable without stopping

```bash
sudo systemctl disable phoenix-heartbeat.timer
```

The resulting state was:

```text
disabled
active
```

This demonstrated that:

- `disable` changes future boot behavior;
- it does not automatically stop a running timer.

### Restore final state

```bash
sudo systemctl enable phoenix-heartbeat.timer
```

Final state:

```text
enabled
active
```

## Journal Verification

The timer and service events were reviewed together:

```bash
journalctl \
  -u phoenix-heartbeat.timer \
  -u phoenix-heartbeat.service \
  --since "20 minutes ago" \
  --no-pager
```

The journal showed:

- manual service execution;
- service execution during boot;
- timer startup;
- timer-triggered service execution;
- timer stop;
- timer restart;
- successful service completion.

## Final Design Cleanup

At one point, both the service and timer were enabled.

This would create two independent automatic execution mechanisms:

```text
boot target → service
timer → service
```

The cleaner final design is:

```text
boot → timer → service → script
```

The service was therefore disabled:

```bash
sudo systemctl disable phoenix-heartbeat.service
```

Final state:

```text
phoenix-heartbeat.service = disabled
phoenix-heartbeat.timer   = enabled
```

The service remains available for manual execution:

```bash
sudo systemctl start phoenix-heartbeat.service
```

The timer remains the only automatic scheduling mechanism.

## Final Verification

The final state was checked with:

```bash
systemctl is-enabled phoenix-heartbeat.service
systemctl is-enabled phoenix-heartbeat.timer
systemctl is-active phoenix-heartbeat.timer
systemctl list-timers phoenix-heartbeat.timer
```

Confirmed result:

```text
service: disabled
timer: enabled
timer state: active
```

## Key Concepts

### Script

The script performs the real task.

```text
/usr/local/bin/phoenix-heartbeat.sh
```

### Service

The service defines how systemd executes the task.

```text
phoenix-heartbeat.service
```

### Timer

The timer defines when the service is activated.

```text
phoenix-heartbeat.timer
```

### `start`

Runs a service or timer now.

### `stop`

Stops a service or timer now.

### `enable`

Configures automatic startup during boot.

### `disable`

Removes automatic startup during boot.

### `enable --now`

Enables automatic startup and starts the unit immediately.

### `daemon-reload`

Reloads systemd unit definitions after unit files are created or changed.

### `active (waiting)`

The timer is active and waiting for the next scheduled activation.

### `inactive (dead)` for a oneshot service

The one-time task completed and no process needs to remain running.

## Practical Applications

The same design can be used for:

- periodic configuration backups;
- filesystem cleanup;
- service health checks;
- certificate expiry checks;
- storage availability checks;
- log rotation helpers;
- report generation;
- monitoring probes;
- notification scripts;
- database maintenance;
- container health verification;
- infrastructure inventory collection.

Example architecture:

```text
backup.timer
    ↓
backup.service
    ↓
/usr/local/bin/backup-configs.sh
```

## Operational Notes

- Test scripts manually before connecting them to systemd.
- Use absolute paths in `ExecStart`.
- Run `daemon-reload` after changing unit files.
- A timer activates a service; it does not replace the service.
- A successful oneshot service does not remain active.
- `enable` and `start` control different aspects of unit behavior.
- Review the journal when scheduled execution does not occur.
- Avoid enabling both a timer and its service unless two boot paths are intentionally required.
- Custom service files belong in `/etc/systemd/system`.
- Custom administrative scripts commonly belong in `/usr/local/bin`.

## Cleanup Procedure

To stop and disable the timer:

```bash
sudo systemctl disable --now phoenix-heartbeat.timer
```

To disable the service:

```bash
sudo systemctl disable phoenix-heartbeat.service
```

To remove the lab files:

```bash
sudo rm /etc/systemd/system/phoenix-heartbeat.timer
sudo rm /etc/systemd/system/phoenix-heartbeat.service
sudo rm /usr/local/bin/phoenix-heartbeat.sh
```

Then reload systemd:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

The heartbeat log may be removed separately:

```bash
sudo rm /var/log/phoenix-heartbeat.log
```

The cleanup commands are documented for recovery and future removal. They were not executed during the completed lab.

## Final Result

| Check | Result |
|---|---|
| Script created | Passed |
| Script permissions verified | Passed |
| Manual script execution | Passed |
| Service unit created | Passed |
| Service recognized by systemd | Passed |
| Manual service execution | Passed |
| Boot persistence tested | Passed |
| Timer unit created | Passed |
| Timer enabled | Passed |
| Timer active | Passed |
| Scheduled execution verified | Passed |
| Journal reviewed | Passed |
| Timer lifecycle tested | Passed |
| Duplicate boot activation removed | Passed |
| Final timer design verified | Passed |

## Lessons Learned

- systemd services and timers have separate responsibilities.
- The service defines execution.
- The timer defines scheduling.
- A script should be tested independently first.
- `inactive (dead)` can represent successful completion.
- `enabled` does not necessarily mean currently active.
- `active` does not necessarily mean enabled for boot.
- Journals provide evidence of actual execution.
- Final configuration cleanup is part of good infrastructure design.
- Simple labs can model real backup, monitoring, and maintenance workflows.

## Outcome

A custom systemd service and timer were successfully created, tested, troubleshot, and verified on `phoenix-linux-01`.

The final implementation uses:

- a custom Bash script;
- a oneshot systemd service;
- a recurring systemd timer;
- a dedicated log file;
- systemd journal verification;
- a clean timer-controlled execution model.