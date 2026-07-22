# Scheduled Tasks and Automation Lab

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Scheduled Tasks and Automation |
| Subtopic | cron, systemd Timers, Troubleshooting, and Safe Recurring Jobs |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Status | Completed and verified |

## Purpose

This runbook documents a practical comparison between traditional `cron` scheduling and `systemd` timers.

The lab covered:

- listing systemd timers;
- understanding timer-to-service relationships;
- reviewing a custom timer and oneshot service;
- verifying enabled and disabled unit states;
- inspecting the cron daemon;
- reading the system-wide crontab;
- creating a user-level cron job;
- defining an explicit cron environment;
- verifying execution through logs;
- capturing a controlled failure;
- restoring the valid job;
- removing the temporary schedule safely.

---

# Environment

```text
Host: phoenix-linux-01
Operating system: Ubuntu Server 24.04.1 LTS
Administrator: igor
Persistent lab storage: /mnt/phoenix-data
```

Existing Project Phoenix automation:

```text
phoenix-heartbeat.timer
phoenix-heartbeat.service
```

Temporary cron output:

```text
/mnt/phoenix-data/cron-lab.log
```

Temporary error output:

```text
/mnt/phoenix-data/cron-error.log
```

---

# Part 1 — Listing systemd Timers

All systemd timers were listed with:

```bash
systemctl list-timers --all
```

Important timers observed included:

```text
phoenix-heartbeat.timer
logrotate.timer
apt-daily.timer
apt-daily-upgrade.timer
fstrim.timer
```

## Timer Output Columns

```text
NEXT
```

The next planned activation.

```text
LEFT
```

The remaining time until the next activation.

```text
LAST
```

The previous activation time.

```text
PASSED
```

The elapsed time since the previous activation.

```text
UNIT
```

The timer unit.

```text
ACTIVATES
```

The service unit started by the timer.

## Examples

```text
phoenix-heartbeat.timer
```

Runs approximately every five minutes.

```text
logrotate.timer
```

Runs the daily log rotation process.

```text
apt-daily.timer
```

Schedules periodic package metadata activity.

Some system timers use randomized delays so that many systems do not contact package repositories simultaneously.

---

# Part 2 — Timer-to-Service Relationship

The Project Phoenix timer was inspected with:

```bash
systemctl cat phoenix-heartbeat.timer
```

Configuration:

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

## Directive Explanation

### `OnBootSec=2min`

Schedules the first activation approximately two minutes after boot.

### `OnUnitActiveSec=5min`

Schedules the next activation five minutes after the associated unit was last activated.

### `Unit=phoenix-heartbeat.service`

Defines the service that the timer starts.

### `WantedBy=timers.target`

Allows the timer to be enabled as part of the standard systemd timer target.

## Responsibility Separation

```text
Timer unit   → defines when
Service unit → defines what
Script       → performs the actual work
```

---

# Part 3 — Reviewing the Service Unit

The related service was inspected with:

```bash
systemctl cat phoenix-heartbeat.service
```

Configuration:

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

## Directive Explanation

### `After=network.target`

Orders the service after basic network initialization.

This does not guarantee internet connectivity.

### `Type=oneshot`

Runs one task and exits.

The service does not remain active as a long-running daemon.

### `ExecStart`

Defines the command or script that performs the work.

```text
/usr/local/bin/phoenix-heartbeat.sh
```

### `WantedBy=multi-user.target`

Allows the service itself to be enabled for normal multi-user startup.

In this design, the service is not enabled directly because the timer activates it.

---

# Part 4 — Enabled State

The units were checked with:

```bash
systemctl is-enabled phoenix-heartbeat.timer
systemctl is-enabled phoenix-heartbeat.service
```

Expected and confirmed model:

```text
phoenix-heartbeat.timer   → enabled
phoenix-heartbeat.service → disabled
```

This is correct for a oneshot service that should only run when activated by its timer.

Enabling both could create unintended duplicate execution paths.

---

# Part 5 — Inspecting the Cron Daemon

The traditional cron service was checked with:

```bash
systemctl status cron
```

Confirmed state:

```text
Loaded: loaded
Enabled: yes
Active: active (running)
```

The service process was:

```text
/usr/sbin/cron
```

The service journal showed actual cron activity, including jobs executed from:

```text
/etc/cron.hourly
```

and periodic `sysstat` data collection.

---

# Part 6 — System-Wide Crontab

The main system crontab was inspected with:

```bash
cat /etc/crontab
```

System-wide cron syntax:

```text
minute hour day-of-month month day-of-week user command
```

Example:

```cron
17 * * * * root cd / && run-parts --report /etc/cron.hourly
```

Meaning:

```text
Minute: 17
Hour: every hour
Day of month: every day
Month: every month
Day of week: every day
User: root
Command: execute valid files in /etc/cron.hourly
```

## System-Wide Cron Directories

Common directories include:

```text
/etc/cron.hourly
/etc/cron.daily
/etc/cron.weekly
/etc/cron.monthly
/etc/cron.d
```

Files in `/etc/crontab` and `/etc/cron.d/` include an explicit user field.

User crontabs do not include that field.

---

# Part 7 — Initial User Crontab State

The current user crontab was checked with:

```bash
crontab -l
```

Initial result:

```text
no crontab for igor
```

This confirmed that no user-level cron schedule existed before the lab.

---

# Part 8 — Creating a User-Level Cron Job

The user crontab was opened with:

```bash
crontab -e
```

A temporary recurring job was added:

```cron
*/2 * * * * /usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1
```

## Schedule Explanation

```text
*/2 * * * *
```

Runs every two minutes.

## Command Explanation

```text
/usr/bin/date --iso-8601=seconds
```

Prints the current timestamp in ISO 8601 format.

The full command path was used because cron has a limited environment.

## Output Redirection

```text
>> /mnt/phoenix-data/cron-lab.log
```

Appends standard output to the log file.

## Error Redirection

```text
2>&1
```

Redirects standard error to the same destination as standard output.

This prevents errors from being lost or sent to local mail unexpectedly.

---

# Part 9 — Verifying Execution

The job produced:

```text
2026-07-22T19:48:01+00:00
```

Subsequent executions appeared at:

```text
19:50
19:52
```

This confirmed that:

- the cron daemon was active;
- the user crontab was installed;
- the schedule was correct;
- the command executed as `igor`;
- the user had permission to write to `/mnt/phoenix-data`;
- output redirection worked.

---

# Part 10 — Verifying the Audit Trail

The cron event was located in `/var/log/syslog`:

```bash
sudo grep "cron-lab.log" /var/log/syslog | tail -n 5
```

Observed event:

```text
(igor) CMD (/usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1)
```

This confirmed:

```text
Scheduler: cron
Execution identity: igor
Command: exact configured command
Audit location: /var/log/syslog
```

Cron output files and system log events provide complementary troubleshooting evidence.

---

# Part 11 — Defining an Explicit Cron Environment

The user crontab was updated to include:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin
```

Final active test configuration:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin

*/2 * * * * /usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1
```

## Why This Matters

Cron does not use the same environment as the user's interactive terminal.

The user normally works with Fish, but the cron job used:

```text
/bin/sh
```

Explicit environment configuration reduces ambiguity.

It also improves:

- reproducibility;
- portability;
- troubleshooting;
- documentation quality.

Even with a defined `PATH`, absolute command paths remain a good operational practice.

---

# Part 12 — Verifying Execution After Environment Changes

After adding `SHELL` and `PATH`, the output remained stable:

```text
2026-07-22T19:48:01+00:00
2026-07-22T19:50:01+00:00
2026-07-22T19:52:01+00:00
```

This proved that the explicit environment settings did not break the scheduled job.

---

# Part 13 — Controlled Failure Test

The valid command was temporarily replaced with a non-existent command:

```cron
*/2 * * * * /usr/bin/nonexistent-phoenix-command >> /mnt/phoenix-data/cron-error.log 2>&1
```

After the next scheduled run, the error file contained:

```text
/bin/sh: 1: /usr/bin/nonexistent-phoenix-command: not found
```

This confirmed that:

- the cron job executed;
- cron used `/bin/sh`;
- the command failed as intended;
- standard error was captured;
- `2>&1` worked correctly;
- the error could be investigated without local mail.

## Troubleshooting Lesson

When an expected cron output file does not appear, first verify the installed configuration:

```bash
crontab -l
```

During this lab, the failure-test line was initially not saved.

The missing output file was therefore caused by configuration state, not by cron itself.

---

# Part 14 — Recovery

The valid command was restored:

```cron
*/2 * * * * /usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1
```

New timestamps appeared:

```text
2026-07-22T19:54:01+00:00
2026-07-22T19:56:01+00:00
```

This confirmed successful recovery after the controlled failure.

The temporary error log was removed:

```bash
rm /mnt/phoenix-data/cron-error.log
```

Its removal was verified.

---

# Part 15 — Cleanup

The recurring test job was removed with:

```bash
crontab -e
```

Only the environment definitions were retained:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin
```

Final user crontab state:

```text
No active scheduled jobs
Explicit shell retained
Explicit PATH retained
```

The final output file metadata was checked:

```bash
stat -c 'Modify: %y  Size: %s bytes' /mnt/phoenix-data/cron-lab.log
```

Final result:

```text
Modify: 2026-07-22 19:56:01 UTC
Size: 130 bytes
```

No later timestamp was added after the schedule was removed.

This confirmed that cleanup was successful.

---

# Cron Syntax Reference

```text
* * * * *
│ │ │ │ │
│ │ │ │ └── day of week
│ │ │ └──── month
│ │ └────── day of month
│ └──────── hour
└────────── minute
```

## Common Examples

Every five minutes:

```cron
*/5 * * * *
```

Every day at 02:30:

```cron
30 2 * * *
```

Every Monday at 09:00:

```cron
0 9 * * 1
```

On the first day of every month at midnight:

```cron
0 0 1 * *
```

Every hour at minute 15:

```cron
15 * * * *
```

---

# Cron and systemd Timer Comparison

| Feature | cron | systemd timer |
|---|---|---|
| Traditional availability | Excellent | systemd systems |
| Configuration | Single cron line | Timer and service units |
| Execution identity | User crontab or user field | `User=` in service |
| Logging | syslog and redirected output | systemd journal |
| Dependencies | Limited | Native unit dependencies |
| Missed-run handling | Limited | `Persistent=true` |
| Random delay | Manual or limited | `RandomizedDelaySec=` |
| Boot-relative schedules | Special macros | Native directives |
| Security controls | Shell and file permissions | systemd sandboxing options |
| Status inspection | Indirect | `systemctl status` |
| Failure status | Command output and logs | Unit failure state |
| Complex services | Less suitable | Strong support |

---

# When to Use cron

Cron is useful for:

- simple recurring commands;
- user-level personal jobs;
- existing legacy automation;
- portable schedules across Unix-like systems;
- small scripts without dependency requirements.

Example:

```cron
0 2 * * * /usr/local/bin/simple-backup.sh
```

---

# When to Use systemd Timers

Systemd timers are useful when the task requires:

- service dependencies;
- structured logging;
- failure status;
- resource limits;
- a dedicated execution user;
- sandboxing;
- boot-relative scheduling;
- missed-run recovery;
- randomized delay;
- integration with other systemd units.

Example architecture:

```text
backup.timer
      ↓
backup.service
      ↓
backup script
```

---

# Safe Automation Principles

## Use Full Paths

Prefer:

```text
/usr/bin/date
/usr/bin/rsync
/usr/local/bin/script.sh
```

Do not assume the interactive shell PATH will exist.

## Define the Environment

For cron:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin
```

For systemd, define required environment values inside the service unit or an environment file.

## Redirect Output

For cron:

```text
>> /path/to/job.log 2>&1
```

For systemd, prefer the journal unless a dedicated application log is required.

## Use the Correct User

Do not schedule every task as `root`.

Use the least privileged identity capable of completing the task.

## Test the Command Manually

Before scheduling a command:

1. run it manually;
2. verify its exit status;
3. verify permissions;
4. verify output;
5. then schedule it.

## Test Failure Behavior

A scheduled task is not complete until failure handling is understood.

Verify:

- where errors appear;
- whether the task retries;
- whether it leaves partial files;
- whether it requires alerting;
- how it is restored.

## Clean Up Temporary Schedules

Lab jobs should not remain active after testing.

Repeated temporary jobs can:

- fill disks;
- duplicate work;
- generate noise;
- consume resources;
- hide real operational issues.

---

# Troubleshooting

## Job Does Not Run

Check the cron service:

```bash
systemctl status cron
```

Check the installed user crontab:

```bash
crontab -l
```

Check system logs:

```bash
sudo grep CRON /var/log/syslog | tail
```

Check the current system time and timezone:

```bash
timedatectl
```

## Output File Does Not Exist

Possible reasons:

- the scheduled time has not arrived;
- the cron line was not saved;
- the parent directory is not writable;
- the command path is incorrect;
- the job writes somewhere else;
- shell quoting is invalid.

## Command Works Manually but Fails in cron

Likely causes:

- different shell;
- restricted PATH;
- missing environment variables;
- relative file paths;
- different working directory;
- unavailable credentials;
- non-interactive execution.

Use:

- absolute command paths;
- absolute file paths;
- explicit environment values;
- redirected output and errors.

## Cron Job Runs as the Wrong User

User crontabs run as the owner of the crontab.

System-wide cron files include a separate user field.

Verify the system log entry:

```text
(username) CMD (...)
```

## systemd Timer Is Active but Service Fails

Check:

```bash
systemctl status example.timer
systemctl status example.service
journalctl -u example.service
```

The timer can be healthy while the service it activates fails.

## Timer Does Not Run After Reboot

Check:

```bash
systemctl is-enabled example.timer
systemctl status example.timer
```

Confirm the timer has an `[Install]` section and is enabled.

---

# Security Considerations

- Do not place passwords, tokens, or private keys directly in crontab lines.
- Do not make scripts writable by unauthorized users.
- Use dedicated service users for recurring system tasks.
- Avoid broad sudo rules for automation.
- Protect output logs.
- Use absolute paths.
- Validate ownership of scheduled scripts.
- Avoid executing commands from world-writable directories.
- Review system and user crontabs regularly.
- Remove obsolete automation.
- Consider systemd sandboxing for sensitive recurring services.

---

# Backup and Recovery Considerations

Scheduled backup jobs must be tested beyond command execution.

A complete backup schedule requires:

- a verified destination;
- sufficient free space;
- retention management;
- failure logging;
- restore testing;
- alerting;
- protection against deletion or compromise.

A successful scheduler status does not prove that the backup data is valid.

System automation configuration should be included in configuration backups:

```text
/etc/systemd/system/
/etc/cron.d/
/etc/crontab
user crontabs
/usr/local/bin/
```

User crontabs can be exported with:

```bash
crontab -l > user-crontab-backup.txt
```

---

# Final Verification Table

| Check | Result |
|---|---|
| systemd timers listed | Passed |
| Heartbeat timer identified | Passed |
| Logrotate timer identified | Passed |
| APT timer identified | Passed |
| Timer unit inspected | Passed |
| Service unit inspected | Passed |
| Timer enabled state verified | Passed |
| Service disabled state verified | Passed |
| Cron daemon verified | Passed |
| System-wide crontab inspected | Passed |
| User crontab initial state checked | Passed |
| Test cron job created | Passed |
| Test output verified | Passed |
| Cron syslog event verified | Passed |
| Explicit shell configured | Passed |
| Explicit PATH configured | Passed |
| Execution after environment change verified | Passed |
| Controlled failure created | Passed |
| Standard error captured | Passed |
| Valid job restored | Passed |
| Recovery verified | Passed |
| Error log removed | Passed |
| Temporary recurring job removed | Passed |
| Final crontab inspected | Passed |
| Final log modification time verified | Passed |

---

# Lessons Learned

- systemd timers and services separate scheduling from execution.
- A timer should normally be enabled while its oneshot service remains disabled.
- Cron remains useful for simple user-level recurring commands.
- System-wide cron files include an explicit user field.
- User crontabs run as the user who owns them.
- Cron uses a limited, non-interactive environment.
- Fish shell settings do not automatically apply to cron.
- Full command paths improve reliability.
- `2>&1` captures standard error with standard output.
- `/var/log/syslog` provides an audit trail for cron activity.
- A missing output file may indicate that the cron entry was never saved.
- Controlled failure tests reveal real troubleshooting behavior.
- Recovery must be verified after restoring the valid configuration.
- Temporary lab schedules must be removed after testing.
- A scheduler starting successfully does not prove that the underlying task completed correctly.

---

# Outcome

The `phoenix-linux-01` system now has a documented and tested understanding of both scheduling models.

The lab demonstrated:

- systemd timer inspection;
- timer and service separation;
- recurring user-level cron execution;
- explicit environment configuration;
- logging and audit verification;
- controlled failure handling;
- successful recovery;
- safe cleanup.

No temporary recurring cron job remains active.