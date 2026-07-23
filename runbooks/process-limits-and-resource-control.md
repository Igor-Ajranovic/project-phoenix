# Process Limits and Resource Control

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Process Limits and Resource Control |
| Subtopic | CPU Priority, Shell Limits, systemd Constraints, Verification, and Cleanup |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Status | Completed and verified |

## Purpose

This runbook documents a practical Linux resource-management lab on `phoenix-linux-01`.

The lab covered:

- CPU load and memory baselining;
- process-count inspection;
- identifying CPU and memory consumers;
- process priority through `nice`;
- safe workload termination through `timeout`;
- shell resource limits through `ulimit`;
- soft and hard limits;
- systemd service resource controls;
- file-descriptor limits;
- memory limits;
- CPU quota enforcement;
- exit-status handling;
- verification through process data and systemd accounting;
- cleanup of the temporary lab unit.

No production service was modified.

---

# Environment

```text
System: phoenix-linux-01
Operating system: Ubuntu Server 24.04.1 LTS
Administrator: igor
Shell session: SSH
Interactive shell resource scope: current session
Service manager: systemd
```

Temporary lab unit:

```text
/etc/systemd/system/phoenix-resource-lab.service
```

The unit was removed after testing.

---

# Part 1 — Establishing a Resource Baseline

The initial system state was checked with:

```bash
uptime
free -h
ps -e --no-headers | wc -l
```

## Observed State

```text
Uptime: 1 day, 14 hours, 56 minutes
Logged-in users: 2
Load average: 0.00, 0.00, 0.00
Processes: 112
```

Memory state:

```text
Total memory: 3.8 GiB
Used memory: 468 MiB
Free memory: 1.9 GiB
Buffer and cache: 1.7 GiB
Available memory: 3.3 GiB
```

Swap state:

```text
Total swap: 3.1 GiB
Used swap: 0 B
```

## Interpretation

The virtual machine was effectively idle.

The load averages represented the previous:

```text
1 minute
5 minutes
15 minutes
```

All three values were `0.00`, indicating no meaningful CPU scheduling pressure.

The most useful memory field for operational analysis was:

```text
available
```

This estimates how much memory can be supplied to new workloads without requiring significant swap usage.

The system had approximately:

```text
3.3 GiB available memory
```

and no active swap use.

---

# Part 2 — Inspecting CPU and Memory Consumers

Processes were sorted by CPU usage:

```bash
ps -eo pid,user,ni,comm,%cpu,%mem \
  --sort=-%cpu | head
```

Processes were then sorted by memory usage:

```bash
ps -eo pid,user,ni,comm,%cpu,%mem \
  --sort=-%mem | head
```

## Important Columns

```text
PID
```

Process identifier.

```text
USER
```

Process owner.

```text
NI
```

Nice value.

```text
COMMAND
```

Executable or process name.

```text
%CPU
```

CPU consumption.

```text
%MEM
```

Percentage of physical memory used.

## Observed State

The system showed no significant CPU usage.

The largest listed memory consumers were still below one percent of total memory.

Examples included:

```text
multipathd
unattended-upgrades
systemd-journald
VS Code Remote Server
```

This confirmed that the VM had sufficient capacity for a controlled resource-management lab.

---

# Part 3 — Understanding Nice Values

Linux process scheduling priority can be influenced through the nice value.

Typical range:

```text
-20 → highest scheduling priority
  0 → normal priority
 19 → lowest scheduling priority
```

Important rule:

```text
Lower nice value  → higher CPU scheduling priority
Higher nice value → lower CPU scheduling priority
```

Nice values do not directly define a CPU percentage.

They influence how the scheduler distributes CPU time when processes compete for CPU resources.

---

# Part 4 — Running a Low-Priority CPU Test

A controlled workload was started:

```bash
nice -n 10 \
  timeout 30s \
  sha256sum /dev/zero \
  >/dev/null &
```

## Command Explanation

### `nice -n 10`

Started the workload with a lower scheduling priority than normal processes.

### `timeout 30s`

Automatically stopped the workload after 30 seconds.

### `sha256sum /dev/zero`

Calculated hashes from the endless stream produced by `/dev/zero`.

This created predictable CPU load.

### `>/dev/null`

Discarded standard output.

### `&`

Started the command in the background.

## Process Verification

The workload was inspected with:

```bash
ps -eo pid,user,ni,comm,%cpu \
  --sort=-%cpu | head
```

Observed result:

```text
PID: 11032
User: igor
Nice value: 10
Command: sha256sum
CPU usage: 99.8%
```

## Interpretation

Although the process had:

```text
NI = 10
```

it still used almost one full CPU core because no other process was competing for that CPU time.

This demonstrated:

```text
nice changes scheduling priority
nice does not impose a fixed CPU usage limit
```

---

# Part 5 — Verifying Automatic Termination

After 30 seconds, the shell reported:

```text
Exit 124
```

The process was then checked:

```bash
ps -p 11032 -o pid,ni,comm,%cpu
```

Only the column header remained.

This confirmed that the process no longer existed.

## Timeout Exit Code

Exit status:

```text
124
```

is the expected result when `timeout` stops a command because the configured duration has elapsed.

The workload did not terminate on its own because `/dev/zero` provides an endless data stream.

---

# Part 6 — Inspecting Shell Resource Limits

Current shell limits were displayed with:

```bash
ulimit -a
```

Important observed values:

```text
Core file size: 0
Open files: 1024
Maximum user processes: 15144
Stack size: 8192 KB
CPU time: unlimited
Virtual memory: unlimited
```

## Core File Size

```text
0
```

Core dump files were disabled for processes started from the current shell session.

## Open Files

```text
1024
```

The active shell soft limit allowed a process to hold up to 1024 file descriptors.

File descriptors include:

- regular files;
- network sockets;
- pipes;
- devices;
- other open kernel objects.

## Maximum User Processes

```text
15144
```

This limit applies to process and thread creation for the user.

## Stack Size

```text
8192 KB
```

The default process stack limit was 8 MiB.

## CPU Time and Virtual Memory

Both were unlimited in the current shell session.

---

# Part 7 — Soft and Hard Limits

The open-file limits were checked separately:

```bash
ulimit -Sn
ulimit -Hn
```

Observed values:

```text
Soft limit: 1024
Hard limit: 1048576
```

## Soft Limit

The soft limit is the currently active limit inherited by processes launched from the shell.

## Hard Limit

The hard limit is the maximum value to which the soft limit can normally be raised by the user.

An unprivileged user can generally:

- lower the soft limit;
- raise the soft limit up to the hard limit;
- not raise the hard limit beyond its configured value.

---

# Part 8 — Temporarily Raising the Soft Limit

The current shell soft limit was changed:

```bash
ulimit -Sn 4096
```

Verification:

```bash
ulimit -Sn
```

Result:

```text
4096
```

## Scope

The change applied only to:

- the current shell session;
- child processes launched from that session.

It did not permanently modify:

- system-wide configuration;
- other SSH sessions;
- systemd service limits;
- future login sessions.

A new login session would normally receive the default configured value again.

---

# Part 9 — Comparing Shell and systemd Limits

The existing heartbeat service was inspected:

```bash
systemctl show phoenix-heartbeat.service \
  -p LimitNOFILE \
  -p LimitNPROC \
  -p MemoryMax \
  -p CPUQuotaPerSecUSec
```

Observed values:

```text
CPUQuotaPerSecUSec=infinity
MemoryMax=infinity
LimitNOFILE=524288
LimitNPROC=15144
```

## Interpretation

The heartbeat service had no explicit CPU or memory ceiling.

Its file-descriptor limit was:

```text
524288
```

This was significantly higher than the initial interactive shell soft limit of:

```text
1024
```

## Scope Difference

```text
ulimit
```

controls the current shell and its child processes.

Systemd directives such as:

```text
LimitNOFILE=
LimitNPROC=
MemoryMax=
CPUQuota=
```

apply directly to a managed service and its cgroup.

---

# Part 10 — Creating a Dedicated Resource-Control Service

A temporary unit was created:

```text
/etc/systemd/system/phoenix-resource-lab.service
```

Initial configuration:

```ini
[Unit]
Description=Project Phoenix resource control lab

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c 'ulimit -n; echo "resource lab completed"'
LimitNOFILE=256
MemoryMax=64M
CPUQuota=25%
```

## Resource Directives

### `LimitNOFILE=256`

Set the service file-descriptor limit to 256.

### `MemoryMax=64M`

Set the service cgroup memory ceiling to 64 MiB.

### `CPUQuota=25%`

Limited the service to approximately 25 percent of one CPU core.

### `Type=oneshot`

The service performed one task and exited.

---

# Part 11 — Loading and Validating the Unit

Systemd was instructed to reload unit files:

```bash
sudo systemctl daemon-reload
```

The unit was validated:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/phoenix-resource-lab.service
```

No validation errors were reported.

This confirmed that the unit syntax was acceptable before execution.

---

# Part 12 — Verifying the File-Descriptor Limit

The service was started:

```bash
sudo systemctl start phoenix-resource-lab.service
```

Its journal was checked:

```bash
journalctl -u phoenix-resource-lab.service \
  -n 10 \
  --no-pager
```

Observed output:

```text
256
resource lab completed
Deactivated successfully
Finished
```

This confirmed that:

```ini
LimitNOFILE=256
```

was active inside the service.

---

# Part 13 — Inspecting Applied systemd Limits

The applied values were inspected:

```bash
systemctl show phoenix-resource-lab.service \
  -p LimitNOFILE \
  -p MemoryMax \
  -p CPUQuotaPerSecUSec
```

Observed values:

```text
CPUQuotaPerSecUSec=250ms
MemoryMax=67108864
LimitNOFILE=256
```

## Memory Conversion

```text
67108864 bytes = 64 MiB
```

## CPU Quota Conversion

Systemd represented the CPU quota as:

```text
250 ms of CPU time per second
```

Calculation:

```text
250 ms / 1000 ms = 25%
```

This confirmed that all requested limits had been loaded.

---

# Part 14 — Testing CPU Quota Enforcement

The service command was changed to:

```ini
ExecStart=/usr/bin/timeout 20s /usr/bin/sha256sum /dev/zero
```

The resource limits remained:

```ini
LimitNOFILE=256
MemoryMax=64M
CPUQuota=25%
```

The service was reloaded and started:

```bash
sudo systemctl daemon-reload
sudo systemctl start phoenix-resource-lab.service
```

While it was running, process usage was inspected:

```bash
ps -eo pid,comm,%cpu,ni \
  --sort=-%cpu | head
```

Observed result:

```text
sha256sum → 25.1% CPU
```

## Comparison

Earlier test using only `nice`:

```text
CPU usage: approximately 99.8%
```

Systemd quota test:

```text
CPU usage: approximately 25.1%
```

This demonstrated:

```text
nice      → relative scheduler priority
CPUQuota  → enforced CPU consumption ceiling
```

---

# Part 15 — Interpreting Service Failure After Timeout

After 20 seconds, the service status showed:

```text
Active: failed
Result: exit-code
status=124
CPU time: 5.025s
```

The service was initially marked failed because the `timeout` command exited with status:

```text
124
```

This was expected behavior from the test command.

## CPU Accounting Verification

The wall-clock runtime was approximately:

```text
20 seconds
```

CPU quota:

```text
25%
```

Expected CPU time:

```text
20 seconds × 25% = 5 seconds
```

Observed systemd CPU accounting:

```text
5.025 seconds
```

This closely matched the expected value and provided strong evidence that the quota was enforced correctly.

---

# Part 16 — Accepting the Expected Exit Status

The unit was updated with:

```ini
SuccessExitStatus=124
```

Final service configuration:

```ini
[Unit]
Description=Project Phoenix resource control lab

[Service]
Type=oneshot
ExecStart=/usr/bin/timeout 20s /usr/bin/sha256sum /dev/zero
SuccessExitStatus=124
LimitNOFILE=256
MemoryMax=64M
CPUQuota=25%
```

## Meaning

`SuccessExitStatus=124` instructed systemd to treat exit code `124` as an accepted successful result for this service.

This was appropriate because the timeout was intentional and represented the planned end of the test.

---

# Part 17 — Resetting Failed State and Retesting

Systemd was reloaded:

```bash
sudo systemctl daemon-reload
```

The previous failed state was cleared:

```bash
sudo systemctl reset-failed \
  phoenix-resource-lab.service
```

The service was started again:

```bash
sudo systemctl start phoenix-resource-lab.service
```

After approximately 20 seconds, its status showed:

```text
Active: inactive (dead)
Deactivated successfully
Finished
Consumed 5.025s CPU time
```

The timeout exit code was now accepted as successful.

This confirmed:

- `CPUQuota=25%` remained effective;
- systemd CPU accounting remained near five seconds;
- `SuccessExitStatus=124` worked;
- the oneshot service completed without a failed state.

---

# Part 18 — Cleanup

The temporary unit was removed:

```bash
sudo rm \
  /etc/systemd/system/phoenix-resource-lab.service
```

Systemd was reloaded:

```bash
sudo systemctl daemon-reload
```

The cleanup was verified:

```bash
systemctl status \
  phoenix-resource-lab.service \
  --no-pager
```

Result:

```text
Unit phoenix-resource-lab.service could not be found.
```

No temporary lab service remained installed.

---

# Nice Versus CPU Quota

| Feature | `nice` | `CPUQuota` |
|---|---|---|
| Purpose | Relative CPU priority | CPU usage ceiling |
| Scope | Process scheduling | Entire service cgroup |
| Fixed CPU percentage | No | Yes |
| Competition required for visible effect | Usually | No |
| Managed through | Process command | systemd service |
| Typical use | Background jobs | Service resource control |

## Example: Nice

```bash
nice -n 10 command
```

The command can still use an entire idle CPU core.

## Example: CPU Quota

```ini
CPUQuota=25%
```

The service is throttled to approximately one quarter of one CPU core.

---

# Shell Limits Versus systemd Limits

| Feature | Shell `ulimit` | systemd |
|---|---|---|
| Scope | Shell and child processes | Service cgroup |
| Persistence | Usually session-only | Unit configuration |
| File descriptors | `ulimit -n` | `LimitNOFILE=` |
| Process count | `ulimit -u` | `LimitNPROC=` or task controls |
| Memory ceiling | Limited shell controls | `MemoryMax=` |
| CPU percentage | Not directly | `CPUQuota=` |
| Logging | Shell-dependent | systemd journal |
| Status inspection | Manual | `systemctl show/status` |

---

# Important systemd Resource Controls

## File Descriptors

```ini
LimitNOFILE=4096
```

Useful for services handling many:

- network connections;
- files;
- sockets;
- pipes.

An excessively low value can cause:

```text
Too many open files
```

## Process and Thread Limits

```ini
LimitNPROC=512
```

or cgroup-oriented task limits such as:

```ini
TasksMax=512
```

These can protect the system from runaway process or thread creation.

## Memory Limit

```ini
MemoryMax=512M
```

Sets a hard memory ceiling for the service cgroup.

When exceeded, the service may be terminated by the cgroup out-of-memory mechanism.

## CPU Quota

```ini
CPUQuota=50%
```

Limits the service to approximately half of one CPU core.

Values above 100 percent can be used on multi-core systems.

Example:

```ini
CPUQuota=200%
```

allows approximately two full CPU cores.

## CPU Weight

```ini
CPUWeight=100
```

Defines relative CPU importance between competing services.

Unlike `CPUQuota`, it does not create a fixed maximum.

---

# Resource-Limit Safety Principles

## Establish a Baseline

Before applying limits, record:

```text
normal CPU use
peak CPU use
normal memory use
peak memory use
open file count
process and thread count
```

## Limit Dedicated Lab Services First

Do not test aggressive limits on production services.

Use a temporary or dedicated service with predictable behavior.

## Change One Limit at a Time

Testing one control at a time makes troubleshooting clearer.

## Verify Under Load

A configured limit is not fully proven until the service is tested under a workload that would exceed it.

## Monitor Service Status

Check:

```bash
systemctl status service
journalctl -u service
systemctl show service
```

## Define Expected Exit Codes

Some tools use non-zero codes for expected operational outcomes.

Use:

```ini
SuccessExitStatus=
```

only when the exit code is understood and intentional.

Do not hide genuine failures by accepting broad ranges of error codes.

## Keep Recovery Simple

Before changing an existing service:

- create a backup of the unit or override;
- know how to remove the override;
- know how to run `daemon-reload`;
- retain console or SSH recovery access.

---

# Troubleshooting

## Service Uses More CPU Than Expected

Check the applied value:

```bash
systemctl show service \
  -p CPUQuotaPerSecUSec
```

Verify that:

- `daemon-reload` was performed;
- the correct unit was modified;
- the workload belongs to the expected cgroup;
- the service was restarted after configuration changes.

## Nice Value Does Not Reduce CPU Percentage

This can be normal.

A nice value changes priority, not a fixed ceiling.

On an otherwise idle CPU, a low-priority process may still use nearly 100 percent of one core.

## Service Fails with Exit Code 124

A command controlled by `timeout` normally returns:

```text
124
```

Determine whether timeout was intentional.

When intentional, a narrowly defined setting may be used:

```ini
SuccessExitStatus=124
```

## Service Reports Too Many Open Files

Inspect:

```bash
systemctl show service -p LimitNOFILE
```

Also inspect the service's actual file-descriptor usage.

Do not increase the limit without investigating possible descriptor leaks.

## Service Is Killed Under Memory Pressure

Check:

```bash
systemctl status service
journalctl -u service
journalctl -k
```

Review:

```ini
MemoryMax=
```

A low memory ceiling may be causing cgroup OOM termination.

## Unit Changes Do Not Apply

Run:

```bash
sudo systemctl daemon-reload
```

Then restart the affected service if required.

## Unit Remains Failed After Correction

Clear the recorded state:

```bash
sudo systemctl reset-failed service
```

This does not correct the underlying configuration by itself.

---

# Security Considerations

Resource controls can improve service isolation and resilience.

They can help reduce the impact of:

- runaway CPU workloads;
- memory leaks;
- file-descriptor leaks;
- fork bombs;
- compromised services;
- poorly configured applications.

However, incorrect limits can create denial-of-service conditions by preventing a legitimate service from obtaining required resources.

Security principles:

- use least privilege;
- apply limits per service;
- test before production deployment;
- monitor failures;
- avoid unexplained unlimited values for exposed services;
- do not hide failure exit codes;
- document every override;
- keep rollback instructions.

---

# Backup and Recovery Considerations

Before modifying a production systemd service, preserve:

```text
original unit file
drop-in override
current systemctl show output
current service status
recent journal entries
```

Prefer drop-in overrides for packaged services:

```bash
sudo systemctl edit service
```

This avoids editing vendor-managed unit files directly.

Typical rollback process:

1. remove or correct the override;
2. run `systemctl daemon-reload`;
3. reset failed state when needed;
4. restart the service;
5. verify service functionality;
6. verify resource values;
7. inspect logs.

---

# Monitoring Considerations

Useful process and service metrics include:

```text
CPU usage
CPU throttling time
memory current
memory maximum
OOM events
open file descriptors
process and thread count
service restart count
failed unit state
```

Tools may include:

- `systemctl`;
- `journalctl`;
- `ps`;
- `top`;
- `htop`;
- `systemd-cgtop`;
- Prometheus Node Exporter;
- cAdvisor;
- Grafana;
- dedicated application monitoring.

Resource limits should be paired with alerts so that throttling or repeated failures are not silently ignored.

---

# Final Verification Table

| Check | Result |
|---|---|
| CPU load baseline collected | Passed |
| Memory baseline collected | Passed |
| Swap usage checked | Passed |
| Process count recorded | Passed |
| CPU consumers inspected | Passed |
| Memory consumers inspected | Passed |
| Nice-value behavior tested | Passed |
| CPU workload safely timed out | Passed |
| Exit code `124` interpreted | Passed |
| Process termination verified | Passed |
| Shell limits inspected | Passed |
| Soft file limit identified | Passed |
| Hard file limit identified | Passed |
| Soft limit temporarily increased | Passed |
| Shell and systemd limits compared | Passed |
| Temporary systemd unit created | Passed |
| Unit syntax verified | Passed |
| File-descriptor limit verified | Passed |
| Memory limit loaded | Passed |
| CPU quota loaded | Passed |
| CPU quota tested under load | Passed |
| CPU use reduced to approximately 25% | Passed |
| systemd CPU accounting verified | Passed |
| Expected timeout status accepted | Passed |
| Service completed successfully | Passed |
| Temporary unit removed | Passed |
| Cleanup verified | Passed |

---

# Lessons Learned

- Load average and CPU percentage describe different aspects of workload.
- Available memory is more useful than the raw free-memory value.
- Swap usage helps indicate memory pressure.
- Nice values change relative scheduling priority.
- Nice does not create a CPU percentage limit.
- `timeout` is useful for safe bounded workload tests.
- Exit status `124` indicates timeout expiration.
- Shell soft and hard limits are different.
- Shell `ulimit` changes normally apply only to the current session and child processes.
- Systemd services can use independent resource limits.
- `LimitNOFILE` controls file descriptors.
- `MemoryMax` controls service-cgroup memory usage.
- `CPUQuota` creates an enforceable CPU ceiling.
- `systemctl show` reveals the applied normalized values.
- CPU accounting can verify quota behavior mathematically.
- Expected non-zero exit statuses can be declared with `SuccessExitStatus`.
- A oneshot service being inactive after completion is normal.
- Temporary lab units should be removed after testing.
- Resource controls must be tested before production use.

---

# Outcome

A complete process and systemd resource-control workflow was successfully tested on `phoenix-linux-01`.

The lab demonstrated:

```text
Low-priority process:
  nice value 10
  approximately 99.8% CPU when uncontested
```

```text
Shell file-descriptor limit:
  soft limit 1024
  hard limit 1048576
  temporary soft limit 4096
```

```text
systemd lab service:
  LimitNOFILE=256
  MemoryMax=64M
  CPUQuota=25%
```

Under load, the service used approximately:

```text
25.1% CPU
```

During a 20-second test, systemd recorded:

```text
5.025 seconds of CPU time
```

This closely matched the expected 25-percent CPU quota.

The temporary service was removed, leaving no persistent lab unit behind.