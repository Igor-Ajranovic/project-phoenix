# System Health and Failed Service Troubleshooting

## Document Information

| Field | Value |
|---|---|
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Hypervisor | Proxmox VE |
| Management IP | `192.168.20.101` |
| Main topics | systemd, journalctl, system health |
| Status | Completed and verified |

## Purpose

This runbook documents a post-update system health verification and a controlled systemd troubleshooting exercise on the `phoenix-linux-01` lab virtual machine.

The objectives were to:

- verify that the system remained healthy after package updates and reboot;
- confirm that networking and SSH were operational;
- inspect system resources;
- create a safe failed-service scenario;
- diagnose the failure using systemd and journal logs;
- clear the failed state without rebooting the server.

## Safety Controls

The exercise was performed on a dedicated lab virtual machine.

The following controls were in place:

- Proxmox console access remained available.
- SSH key-based access had already been verified.
- The exercise did not modify SSH, networking, storage, or production services.
- A transient systemd unit was used instead of a permanent service file.
- The test service intentionally executed `/bin/false`.
- The failed state was removed after the exercise.

## Post-Update Failed Service Check

The first health check was:

```bash
systemctl --failed
```

Confirmed result:

```text
0 loaded units listed.
```

This showed that no systemd units had failed after the package upgrade and reboot.

## Current Boot Error Check

Errors from the current boot were reviewed with:

```bash
journalctl -p err -b
```

Confirmed result:

```text
-- No entries --
```

This showed that the journal contained no messages at the `error` priority level for the current boot.

## Resource Verification

The following commands were used:

```bash
free -h
df -h /
uptime
```

Observed state:

```text
Memory total: approximately 3.8 GiB
Memory used: approximately 406 MiB
Memory available: approximately 3.4 GiB
Swap used: 0 B
Root filesystem usage: 50%
Root filesystem available: approximately 7.0 GB
System load: very low
```

The system had sufficient available memory, disk capacity, and no meaningful CPU load.

## Network and SSH Verification

The following services were checked:

```bash
systemctl is-active systemd-networkd systemd-resolved ssh
```

Confirmed result:

```text
active
active
active
```

The network interfaces were checked with:

```bash
ip -brief address
```

Confirmed relevant state:

```text
ens18 UP 192.168.20.101/24
```

This confirmed that:

- the main network interface was operational;
- the fixed management IP remained assigned;
- systemd-networkd was active;
- systemd-resolved was active;
- the SSH service was active.

## SSH Listening Socket Check

The SSH listening socket was verified with:

```bash
sudo ss -ltnp | grep ':22'
```

The result confirmed that SSH was listening on TCP port `22`.

The SSH service state was inspected with:

```bash
systemctl show ssh \
  --property=LoadState,ActiveState,SubState,MainPID
```

Confirmed result:

```text
MainPID=835
LoadState=loaded
ActiveState=active
SubState=running
```

This showed that the SSH unit was correctly loaded, active, running, and associated with a valid main process.

## Controlled Failure Scenario

A transient systemd service was created:

```bash
sudo systemd-run --unit=phoenix-failure-lab /bin/false
```

The command returned:

```text
Running as unit: phoenix-failure-lab.service
```

The service was intentionally designed to fail because `/bin/false` always exits with a non-zero status.

## Failed Unit Detection

The failed unit was detected with:

```bash
systemctl --failed
```

Confirmed result:

```text
phoenix-failure-lab.service loaded failed failed /bin/false
```

The unit was:

- loaded successfully;
- marked as failed;
- associated with the expected test command.

## Service Status Analysis

The service was inspected with:

```bash
systemctl status phoenix-failure-lab --no-pager
```

Relevant result:

```text
Loaded: loaded
Transient: yes
Active: failed
Result: exit-code
ExecStart=/bin/false
status=1/FAILURE
```

This confirmed that the service definition itself was valid.

The failure was caused by the process returning exit status `1`.

## Journal Analysis

Unit-specific logs were inspected with:

```bash
journalctl -u phoenix-failure-lab --no-pager
```

Relevant result:

```text
Started phoenix-failure-lab.service - /bin/false.
Main process exited, code=exited, status=1/FAILURE
Failed with result 'exit-code'.
```

The journal confirmed the same cause reported by `systemctl status`.

## Clearing the Failed State

The failed state was cleared with:

```bash
sudo systemctl reset-failed phoenix-failure-lab
```

This command did not restart the service.

It only removed the failed state from systemd's runtime tracking.

The result was verified with:

```bash
systemctl --failed
```

Confirmed result:

```text
0 loaded units listed.
```

## Troubleshooting Workflow

The exercise demonstrated the following workflow:

```text
Detect
Inspect service status
Inspect service-specific logs
Identify the exit code
Understand the failure cause
Clear the failed state
Verify system health
```

## Interpretation Guide

### Loaded but failed

Example:

```text
LoadState=loaded
ActiveState=failed
```

Meaning:

The unit definition was loaded successfully, but its process failed during execution.

### Active and running

Example:

```text
LoadState=loaded
ActiveState=active
SubState=running
```

Meaning:

The unit is loaded and its main process is currently running.

### Exit status 1

Example:

```text
status=1/FAILURE
```

Meaning:

The process exited with a generic non-zero error code.

The exit code alone does not always explain the exact cause, so the service logs must also be reviewed.

### Transient unit

Example:

```text
Transient: yes
```

Meaning:

The unit was created dynamically at runtime and is not backed by a permanent service file in `/etc/systemd/system`.

This makes transient units useful for safe lab exercises.

## Common Diagnostic Commands

### List failed units

```bash
systemctl --failed
```

### Show service status

```bash
systemctl status SERVICE_NAME --no-pager
```

### Show selected service properties

```bash
systemctl show SERVICE_NAME \
  --property=LoadState,ActiveState,SubState,MainPID
```

### View logs for one service

```bash
journalctl -u SERVICE_NAME --no-pager
```

### View errors from the current boot

```bash
journalctl -p err -b
```

### Clear a failed state

```bash
sudo systemctl reset-failed SERVICE_NAME
```

## Important Operational Notes

- `systemctl reset-failed` does not repair the underlying problem.
- It only clears the recorded failed state.
- A service should not be restarted repeatedly without understanding the cause.
- `systemctl status` provides a summary.
- `journalctl -u` usually provides more detailed event history.
- Exit codes should be interpreted together with logs.
- A loaded service can still fail during execution.
- A running service can still have application-level problems, so service state is only one part of health verification.
- Resource, network, port, and application checks should be combined.

## Final Result

| Check | Result |
|---|---|
| Failed systemd units before exercise | None |
| Current boot errors | None |
| Memory state | Healthy |
| Swap use | None |
| Root filesystem usage | 50% |
| System load | Low |
| Network services | Active |
| SSH service | Active |
| SSH port | Listening |
| Controlled failure created | Yes |
| Failure detected | Yes |
| Exit code identified | Yes |
| Unit logs reviewed | Yes |
| Failed state cleared | Yes |
| Failed units after cleanup | None |

## Lessons Learned

- A successful boot does not automatically prove that every service is healthy.
- `systemctl --failed` provides a fast first-level health check.
- `journalctl -p err -b` helps identify serious issues from the current boot.
- Memory, disk, load, networking, and service checks should be combined.
- `systemctl status` identifies the service state and recent events.
- `journalctl -u` provides service-specific log history.
- A failed process can be distinguished from a failed unit-loading problem.
- Transient units provide a safe way to practice systemd troubleshooting.
- Clearing a failed state is not the same as repairing a service.
- Final verification is required after every troubleshooting action.

## Outcome

The `phoenix-linux-01` system passed the post-update health checks.

A controlled failed-service scenario was successfully created, diagnosed, and cleaned up.

The exercise demonstrated practical skills in:

- systemd service inspection;
- journal analysis;
- process exit-code interpretation;
- system resource verification;
- SSH and network validation;
- safe troubleshooting workflows;
- failed-state cleanup;
- infrastructure documentation.