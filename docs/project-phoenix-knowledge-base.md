---

# Linux Administration Lab — SSH Security and Remote Administration

## Lab Summary

A dedicated Ubuntu Server virtual machine named `phoenix-linux-01` was deployed in the Proxmox HomeLab environment.

The system was configured for secure SSH administration using an existing HomeLab Ed25519 key. Password-based SSH authentication and direct root SSH access were disabled after successful public-key authentication tests.

## Environment

| Component | Configuration |
|---|---|
| Hypervisor | Proxmox VE |
| Virtual machine | `phoenix-linux-01` |
| Operating system | Ubuntu Server 24.04.1 LTS |
| Management IP | `192.168.20.101` |
| Network | Servers network |
| Administrative user | `igor` |
| SSH port | `22` |
| SSH key type | Ed25519 |

## Engineering Workflow

The lab followed a controlled infrastructure change process:

1. Verify the system identity and network address.
2. Create an Omada Controller backup.
3. Configure a fixed DHCP reservation.
4. Verify the SSH service and listening sockets.
5. Deploy the existing HomeLab public key.
6. Confirm key-based authentication.
7. Inspect server-side SSH file permissions.
8. Configure a local SSH alias.
9. Keep two active recovery sessions open.
10. Create a dedicated SSH hardening drop-in.
11. Validate configuration syntax.
12. Reload SSH without terminating active sessions.
13. Verify the effective configuration.
14. Test a new key-based connection.
15. Confirm that password-only authentication is rejected.
16. Document the configuration and recovery procedure.

## Final SSH Configuration

```ssh
# /etc/ssh/sshd_config.d/10-phoenix-hardening.conf

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Effective configuration:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

## Validation Results

### Successful key-based connection

```bash
ssh phoenix-linux-01
```

Result:

- connection successful;
- correct user confirmed;
- correct hostname confirmed;
- no remote account password required.

### Rejected password-only connection

```bash
ssh -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  igor@192.168.20.101
```

Result:

```text
Permission denied (publickey).
```

## Troubleshooting Scenario

The first hardening file was named:

```text
99-phoenix-hardening.conf
```

Password authentication unexpectedly remained enabled.

Investigation showed that cloud-init had created:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

with:

```ssh
PasswordAuthentication yes
```

The custom hardening file was renamed to:

```text
10-phoenix-hardening.conf
```

After syntax validation and an SSH reload, the intended effective configuration became active.

## Lessons Learned

- Effective configuration must be verified rather than assumed.
- Configuration file order can affect the final SSH settings.
- Cloud-init-managed files should not be modified without understanding their ownership and lifecycle.
- A dedicated configuration drop-in is easier to audit and recover than direct modification of the main SSH configuration.
- Existing sessions and console access should remain available during authentication changes.
- Positive tests confirm that intended access works.
- Negative tests confirm that prohibited access no longer works.
- GUI failure should not block administration when a reliable terminal workflow is available.
- Network configuration changes should be preceded by a current controller backup.

## Portfolio Evidence

This lab demonstrates practical experience with:

- Ubuntu Server administration;
- OpenSSH server management;
- Ed25519 public-key authentication;
- SSH client aliases;
- systemd service management;
- socket and port verification;
- Linux ownership and permissions;
- configuration drop-in files;
- configuration precedence troubleshooting;
- secure change implementation;
- rollback and recovery planning;
- infrastructure documentation.

## Related Runbook

See:

```text
runbooks/ssh-hardening-phoenix-linux-01.md
```
---

# Linux Administration Lab — Safe System Updates and Recovery

## Lab Summary

A controlled operating system update was completed on the `phoenix-linux-01` Ubuntu Server virtual machine.

The workflow included a pre-change snapshot, system health checks, package review, upgrade simulation, controlled installation, reboot, and post-update verification.

## Engineering Workflow

The update followed this sequence:

1. Create a Proxmox snapshot.
2. Verify system load and free disk space.
3. Refresh APT repository metadata.
4. Review available package updates.
5. Simulate the upgrade.
6. Confirm that no packages will be removed.
7. Install the updates.
8. Review restarted and deferred services.
9. Confirm that a reboot is required.
10. Perform a controlled reboot.
11. Reconnect through SSH.
12. Verify system uptime and reboot state.
13. Confirm that networking and SSH remained functional.
14. Retain the snapshot temporarily as a rollback point.
15. Document the procedure and recovery plan.

## Update Summary

The simulated upgrade reported:

```text
111 upgraded, 8 newly installed, 0 to remove and 0 not upgraded.
```

The package installation completed successfully.

The system required a reboot because some services and user sessions were still using outdated binaries.

## Post-Reboot Validation

After reboot, the following conditions were confirmed:

- the VM booted successfully;
- the fixed IP address remained available;
- SSH key-based access worked;
- the SSH alias remained functional;
- networking returned normally;
- no additional reboot was required.

## Recovery Controls

The primary rollback mechanism was the Proxmox snapshot:

```text
before-system-update
```

The Proxmox console remained available as an independent recovery path if SSH or networking failed.

The snapshot will be removed after the system has remained stable for an appropriate verification period.

## Lessons Learned

- Updates should be inspected before installation.
- Upgrade simulation helps identify removals and held packages.
- Snapshots provide a fast rollback mechanism for lab virtual machines.
- Snapshots are not substitutes for independent backups.
- Service restarts may not update all running processes.
- A reboot can be part of the normal update workflow.
- SSH and network access must be tested after reboot.
- Post-change validation is part of the change itself.
- Temporary recovery snapshots should have a defined cleanup point.

## Portfolio Evidence

This lab demonstrates practical experience with:

- Ubuntu package management;
- APT repository metadata;
- upgrade simulation;
- Proxmox snapshots;
- capacity checks;
- change risk assessment;
- controlled server reboot;
- SSH post-reboot validation;
- rollback planning;
- infrastructure runbook creation.

## Related Runbook

```text
runbooks/safe-system-update-phoenix-linux-01.md
```

---

# Linux Administration Lab — System Health and Failed Service Troubleshooting

## Lab Summary

A post-update system health verification and controlled systemd troubleshooting exercise were completed on `phoenix-linux-01`.

The system was checked for failed units, boot errors, resource pressure, network problems, and SSH availability.

A transient systemd unit was then created to simulate a service failure in a safe and reproducible way.

## Health Verification

The following checks were completed:

```bash
systemctl --failed
journalctl -p err -b
free -h
df -h /
uptime
systemctl is-active systemd-networkd systemd-resolved ssh
ip -brief address
sudo ss -ltnp | grep ':22'
```

Confirmed state:

- no failed systemd units;
- no journal errors from the current boot;
- low memory use;
- no swap use;
- root filesystem at approximately 50%;
- low system load;
- networking active;
- DNS resolution service active;
- SSH active and listening;
- fixed management IP preserved.

## Controlled Failure Lab

A transient unit was created:

```bash
sudo systemd-run --unit=phoenix-failure-lab /bin/false
```

The unit entered a failed state because `/bin/false` returned exit status `1`.

The failure was detected with:

```bash
systemctl --failed
```

The service state was inspected with:

```bash
systemctl status phoenix-failure-lab --no-pager
```

The service-specific journal was reviewed with:

```bash
journalctl -u phoenix-failure-lab --no-pager
```

The logs confirmed:

```text
Main process exited, code=exited, status=1/FAILURE
Failed with result 'exit-code'.
```

## Cleanup

The failed state was cleared with:

```bash
sudo systemctl reset-failed phoenix-failure-lab
```

Final verification:

```bash
systemctl --failed
```

Result:

```text
0 loaded units listed.
```

## Troubleshooting Model

The exercise followed this troubleshooting sequence:

```text
Detect
Inspect
Correlate status and logs
Identify the failure type
Apply a safe corrective action
Verify the result
```

## Lessons Learned

- Service health must be verified after updates and reboots.
- `systemctl --failed` is an effective first diagnostic command.
- `systemctl status` and `journalctl -u` provide complementary information.
- A loaded service can still fail during execution.
- Exit codes help classify failures but must be reviewed together with logs.
- `systemctl reset-failed` clears the state but does not fix the original cause.
- Transient units are useful for safe and reproducible troubleshooting labs.
- A complete health check should include services, logs, resources, networking, and listening ports.

## Portfolio Evidence

This lab demonstrates practical experience with:

- systemd service management;
- system health verification;
- journalctl filtering;
- Linux resource inspection;
- SSH service validation;
- TCP listening socket inspection;
- transient systemd units;
- process exit-code analysis;
- failed-state cleanup;
- structured troubleshooting methodology.

## Related Runbook

```text
runbooks/system-health-and-failed-service-troubleshooting.md
```
---

# Linux Administration Lab — systemd Services and Timers

## Lab Summary

A custom systemd service and recurring timer were created on `phoenix-linux-01`.

The lab demonstrated how Linux can execute a custom administrative script manually, during boot, and on a recurring schedule.

## Components

```text
/usr/local/bin/phoenix-heartbeat.sh
/etc/systemd/system/phoenix-heartbeat.service
/etc/systemd/system/phoenix-heartbeat.timer
/var/log/phoenix-heartbeat.log
```

## Execution Model

```text
phoenix-heartbeat.timer
        ↓
phoenix-heartbeat.service
        ↓
phoenix-heartbeat.sh
        ↓
phoenix-heartbeat.log
```

The timer defines when execution occurs.

The service defines how execution occurs.

The script performs the actual task.

## Script

The script writes an ISO 8601 timestamp and heartbeat message to a dedicated log file:

```bash
#!/usr/bin/env bash

echo "$(date --iso-8601=seconds) phoenix-linux-01 heartbeat" >> /var/log/phoenix-heartbeat.log
```

The script was tested manually before systemd integration.

## Service Unit

The service uses:

```ini
Type=oneshot
ExecStart=/usr/local/bin/phoenix-heartbeat.sh
```

A successful oneshot service completes and returns to:

```text
inactive (dead)
```

This is expected behavior.

## Timer Unit

The timer uses:

```ini
OnBootSec=2min
OnUnitActiveSec=5min
Unit=phoenix-heartbeat.service
```

The timer executes the service approximately two minutes after boot and every five minutes after activation.

## Lifecycle Testing

The following states were tested:

```text
enabled + active
enabled + inactive
disabled + active
```

This demonstrated the difference between:

```text
start and stop
enable and disable
runtime state and boot state
```

## Final Design

The service was disabled from direct boot execution.

The timer remains enabled and active.

Final architecture:

```text
boot
  ↓
phoenix-heartbeat.timer
  ↓
phoenix-heartbeat.service
  ↓
phoenix-heartbeat.sh
```

Final state:

```text
phoenix-heartbeat.service = disabled
phoenix-heartbeat.timer   = enabled and active
```

## Validation

Execution was verified through:

```bash
systemctl status
systemctl is-active
systemctl is-enabled
systemctl list-timers
journalctl
tail
```

The heartbeat log confirmed successful:

- manual execution;
- service execution;
- boot execution;
- timer-triggered execution.

## Lessons Learned

- A script performs the real work.
- A service defines how systemd runs the script.
- A timer defines when the service runs.
- `start` controls the current runtime state.
- `enable` controls future boot behavior.
- A oneshot service does not remain active after successful completion.
- Timers provide a structured alternative to cron.
- Journal logs provide proof of scheduled execution.
- Duplicate automatic execution paths should be removed unless intentionally required.

## Practical Applications

This model can be reused for:

- backups;
- health checks;
- cleanup jobs;
- monitoring probes;
- report generation;
- certificate checks;
- maintenance automation;
- infrastructure notifications.

## Portfolio Evidence

This lab demonstrates practical experience with:

- custom Bash scripts;
- Linux filesystem conventions;
- executable permissions;
- custom systemd service units;
- oneshot services;
- systemd timers;
- boot targets;
- unit enablement;
- scheduled automation;
- journal inspection;
- service lifecycle management;
- configuration cleanup.

## Related Runbook

```text
runbooks/systemd-service-and-timer-lab.md
```
---

# Storage, LVM, and Persistent Mounts

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Storage and Filesystems |
| Subtopic | LVM Expansion, QEMU Guest Agent, and Persistent UUID-Based Mounts |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A complete storage administration lab was performed on `phoenix-linux-01`.

The work included:

- inspection of the existing LVM layout;
- online root filesystem expansion;
- QEMU Guest Agent installation;
- creation of a dedicated lab disk;
- GPT partitioning;
- ext4 filesystem creation;
- temporary and persistent mounting;
- ownership configuration;
- `/etc/fstab` validation;
- reboot persistence testing.

## Existing Storage Layout

The VM originally had:

```text
32 GiB virtual disk
30 GiB LVM partition
15 GiB logical volume
15 GiB free inside the volume group
```

The root filesystem was stored on:

```text
/dev/mapper/ubuntu--vg-ubuntu--lv
```

and mounted on:

```text
/
```

## LVM Expansion

A Proxmox snapshot was created before the change:

```text
before-lvm-expansion
```

The logical volume and ext4 filesystem were expanded with:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Final state:

```text
Root logical volume: 30G
Root filesystem: 30G
Available space: approximately 22G
Volume group free space: 0
```

The expansion was completed online without reboot.

## QEMU Guest Agent

The Proxmox snapshot process initially reported that the guest agent was configured but not running.

The package was installed with:

```bash
sudo apt install qemu-guest-agent
```

The agent became active:

```text
Active: active (running)
```

Proxmox then displayed the VM IP addresses:

```text
192.168.20.101
fe80::be24:11ff:febd:df49
```

## Dedicated Data Disk

A separate 4 GiB virtual disk was attached as:

```text
/dev/sdb
```

A GPT partition was created:

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart primary ext4 0% 100%
```

The resulting partition was:

```text
/dev/sdb1
```

An ext4 filesystem was created:

```bash
sudo mkfs.ext4 -L phoenix-data /dev/sdb1
```

Filesystem identity:

```text
Label: phoenix-data
UUID: 44913ffa-70d2-490c-8397-5f17fd2f6db6
```

## Mount Point

The mount point was created:

```bash
sudo mkdir -p /mnt/phoenix-data
```

The filesystem was mounted manually:

```bash
sudo mount /dev/sdb1 /mnt/phoenix-data
```

The mount was verified with:

```bash
findmnt /mnt/phoenix-data
df -h /mnt/phoenix-data
```

## Ownership

The filesystem root was initially owned by `root`.

Ownership was changed:

```bash
sudo chown igor:igor /mnt/phoenix-data
```

Write access was confirmed:

```bash
touch /mnt/phoenix-data/test-file.txt
```

The resulting file was owned by:

```text
igor:igor
```

## Persistent Mount

The existing `/etc/fstab` file was backed up:

```bash
sudo cp /etc/fstab /etc/fstab.backup-before-phoenix-data
```

The following entry was added:

```fstab
UUID=44913ffa-70d2-490c-8397-5f17fd2f6db6 /mnt/phoenix-data ext4 defaults 0 2
```

The configuration was validated with:

```bash
sudo findmnt --verify --verbose
sudo systemctl daemon-reload
sudo findmnt --verify
```

The only remaining warning concerned the standard Ubuntu swap file.

## Mount Testing

The persistent mount was tested without reboot:

```bash
sudo umount /mnt/phoenix-data
sudo mount -a
findmnt /mnt/phoenix-data
```

The mount returned successfully.

The VM was then rebooted.

After reboot:

```bash
findmnt /mnt/phoenix-data
ls -l /mnt/phoenix-data
```

confirmed that:

- the filesystem mounted automatically;
- the test file remained present;
- ownership remained correct;
- the persistent mount worked.

## Storage Model

```text
disk
  ↓
partition
  ↓
filesystem
  ↓
UUID
  ↓
/etc/fstab
  ↓
mount point
  ↓
persistent access after reboot
```

LVM adds additional layers:

```text
disk
  ↓
partition
  ↓
LVM physical volume
  ↓
volume group
  ↓
logical volume
  ↓
filesystem
  ↓
mount point
```

## Lessons Learned

- Free disk space and free LVM space are not the same.
- Logical volume expansion and filesystem expansion are separate operations.
- ext4 can be expanded while mounted.
- New filesystems normally belong to `root`.
- Ownership must match the intended user or service.
- UUID-based mounts are more reliable than `/dev/sdX` names.
- Manual mounts are temporary.
- `/etc/fstab` makes mounts persistent.
- `/etc/fstab` must be validated before reboot.
- `mount -a` is a safe pre-reboot test.
- QEMU Guest Agent improves Proxmox integration and snapshot consistency.

## Portfolio Evidence

This lab demonstrates practical experience with:

- Linux block-device inspection;
- LVM volume groups and logical volumes;
- online ext4 expansion;
- Proxmox snapshots;
- QEMU Guest Agent;
- GPT partitioning;
- ext4 filesystem creation;
- filesystem labels and UUIDs;
- mount points;
- ownership and permissions;
- `/etc/fstab`;
- persistent mounts;
- storage recovery planning.

## Related Runbook

```text
runbooks/storage-lvm-and-persistent-mounts.md
```
---

# Service Accounts and Shared Directory Permissions

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Permissions in Real Service Scenarios |
| Subtopic | Service Accounts, Shared Groups, setgid, umask, and Least Privilege |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A dedicated Linux service account and shared service-data directory were configured on `phoenix-linux-01`.

The lab demonstrated how a service and an authorized administrator can safely share storage without running the service as `root` or granting write access to unrelated users.

## Service Account

The service account was created with:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  phoenixsvc
```

Confirmed properties:

```text
System UID
Primary group: phoenixsvc
No created home directory
Interactive login disabled
```

## Service Data Directory

The directory was created:

```text
/mnt/phoenix-data/service-data
```

Ownership:

```text
phoenixsvc:phoenixsvc
```

The service account successfully created:

```text
service-test.txt
```

## Administrator Access

The administrator was added to the service group:

```bash
sudo usermod -aG phoenixsvc igor
```

A new SSH session was opened to refresh group membership.

Group membership alone did not initially allow writing because the directory did not grant group write permission.

The directory was changed to:

```bash
sudo chmod 775 /mnt/phoenix-data/service-data
```

The administrator could then write through the `phoenixsvc` group.

## setgid

The shared directory was configured with setgid:

```bash
sudo chmod 2775 /mnt/phoenix-data/service-data
```

Final display:

```text
drwxrwsr-x phoenixsvc phoenixsvc service-data
```

New files created by `igor` then inherited:

```text
group: phoenixsvc
```

while retaining:

```text
owner: igor
```

## umask

The administrator's umask was:

```text
0002
```

Resulting file permissions:

```text
664 → rw-rw-r--
```

Resulting directory permissions:

```text
775 → rwxrwxr-x
```

The combination of setgid and umask `0002` produced consistent shared-group access.

## Cross-Account Test

A subdirectory created by `igor` inherited:

```text
owner: igor
group: phoenixsvc
setgid: enabled
```

The service account successfully created a file inside it.

This proved that both the administrator and service account could work inside the same directory tree.

## Negative Access Test

The user `nobody` attempted to write to the service directory.

Result:

```text
Permission denied
```

This confirmed least-privilege enforcement.

## Final State

```text
Directory: /mnt/phoenix-data/service-data
Owner: phoenixsvc
Group: phoenixsvc
Mode: 2775
```

Access:

```text
phoenixsvc → read/write
igor       → read/write through group
nobody     → no write access
```

## Lessons Learned

- Services should use dedicated non-login accounts.
- Administrators should receive access through groups.
- Group membership and directory permissions must both allow writing.
- Supplementary group changes may require a new login session.
- setgid preserves shared group ownership.
- umask controls default write permissions.
- File ownership and group ownership are separate.
- Positive and negative access tests are both necessary.
- Least privilege must be implemented and verified.

## Practical Applications

This model is useful for:

- Docker bind mounts;
- application data;
- database backups;
- shared logs;
- reverse-proxy files;
- monitoring data;
- scheduled backup directories;
- service-generated reports.

## Portfolio Evidence

This lab demonstrates practical experience with:

- Linux system users;
- non-login service accounts;
- supplementary groups;
- ownership and permission management;
- `chmod`;
- `chown`;
- setgid directories;
- umask;
- shared storage design;
- least-privilege validation;
- positive and negative permission testing.

## Related Runbook

```text
runbooks/service-user-and-shared-directory-permissions.md
```
---

# Logging and Log Rotation

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Logging and Log Rotation |
| Subtopic | journald, Traditional Log Files, Rotation, Retention, and Compression |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A complete logging and log rotation workflow was implemented and tested on `phoenix-linux-01`.

The lab used the existing Project Phoenix heartbeat log:

```text
/var/log/phoenix-heartbeat.log
```

The exercise covered traditional log files, persistent journal storage, log metadata, inode behavior, custom logrotate configuration, compression, retention, automatic scheduling, and state tracking.

## Linux Logging Locations

The main traditional log directory is:

```text
/var/log
```

Important examples observed during the lab included:

```text
/var/log/auth.log
/var/log/syslog
/var/log/kern.log
/var/log/dpkg.log
/var/log/apt/
/var/log/journal/
/var/log/phoenix-heartbeat.log
```

The directory:

```text
/var/log/journal/
```

contains persistent binary journal data managed by `systemd-journald`.

Traditional text logs can be read with tools such as:

```text
cat
less
tail
grep
```

Journal data is normally inspected with:

```text
journalctl
```

## Rotated Log Naming

Typical rotation patterns include:

```text
application.log
application.log.1
application.log.2.gz
application.log.3.gz
```

Meaning:

```text
application.log      → active log
application.log.1    → newest rotated log
application.log.2.gz → older compressed log
```

## Custom Log Metadata

Before rotation, the heartbeat log had:

```text
Owner: root
Group: root
Mode: 0644
Inode: 786837
Size: approximately 27 KB
```

The inode was recorded so that the rotation method could be verified later.

## Dedicated Logrotate Rule

The following configuration was created:

```text
/etc/logrotate.d/phoenix-heartbeat
```

Configuration:

```conf
/var/log/phoenix-heartbeat.log {
    su root root
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
}
```

## Directive Summary

```text
su root root
```

Uses an explicit user and group for rotation.

```text
daily
```

Makes the log eligible for rotation once per day.

```text
rotate 7
```

Retains seven rotated copies.

```text
compress
```

Compresses older rotated logs with gzip.

```text
delaycompress
```

Leaves the newest rotated log uncompressed for one additional cycle.

```text
missingok
```

Does not fail if the log file is missing.

```text
notifempty
```

Does not rotate an empty file.

```text
create 0644 root root
```

Creates a new active log with predictable ownership and permissions.

## Validation and Troubleshooting

The configuration was first tested with:

```bash
sudo logrotate -d /etc/logrotate.d/phoenix-heartbeat
```

The initial test detected:

```text
parent directory has insecure permissions
```

The parent directory `/var/log` was group-writable by `syslog`.

The correct solution was to add:

```conf
su root root
```

to the dedicated rule.

The permissions of `/var/log` were not changed.

After the correction, debug validation completed without an error.

## Forced Rotation Test

A controlled rotation was performed with:

```bash
sudo logrotate -f /etc/logrotate.d/phoenix-heartbeat
```

Result:

```text
phoenix-heartbeat.log   → new empty active log
phoenix-heartbeat.log.1 → previous log content
```

The new active log retained:

```text
Owner: root
Group: root
Mode: 0644
```

## Inode Verification

After rotation:

```text
phoenix-heartbeat.log.1 → inode 786837
phoenix-heartbeat.log   → inode 786745
```

The original inode moved to the rotated file.

A new inode was created for the active log.

This confirmed a rename-and-create rotation model rather than `copytruncate`.

## Continued Logging Test

The heartbeat service was manually started after rotation:

```bash
sudo systemctl start phoenix-heartbeat.service
```

A new heartbeat entry appeared in the new active log.

This proved that:

- the active log was recreated correctly;
- the service could continue writing;
- ownership and permissions were valid;
- the rotated data remained preserved.

## Compression Test

A second forced rotation produced:

```text
phoenix-heartbeat.log
phoenix-heartbeat.log.1
phoenix-heartbeat.log.2.gz
```

This confirmed that:

- the newest rotated log remained uncompressed;
- the older rotated log was compressed;
- `delaycompress` behaved as expected.

The compressed log was successfully read with:

```bash
sudo zcat /var/log/phoenix-heartbeat.log.2.gz
```

## Automatic Scheduling

The operating system uses:

```text
logrotate.timer
```

to start:

```text
logrotate.service
```

The timer was confirmed as:

```text
enabled
active (waiting)
```

The service is an oneshot unit.

After a successful run, this state is normal:

```text
inactive (dead)
```

## Execution Flow

```text
logrotate.timer
        ↓
logrotate.service
        ↓
/etc/logrotate.conf
        ↓
/etc/logrotate.d/phoenix-heartbeat
        ↓
rotation, retention, and compression
```

## State Tracking

The logrotate state file is:

```text
/var/lib/logrotate/status
```

The heartbeat entry recorded:

```text
"/var/log/phoenix-heartbeat.log" 2026-7-22-19:31:15
```

This timestamp allows logrotate to determine whether the configured interval has passed.

## Important Distinctions

```text
logrotate.timer
```

Controls when logrotate runs.

```text
daily
```

Defines the minimum rotation interval.

```text
/var/lib/logrotate/status
```

Tracks the last rotation time.

```text
-f
```

Forces rotation regardless of the normal interval.

## Final State

```text
Managed log:
  /var/log/phoenix-heartbeat.log

Rule:
  /etc/logrotate.d/phoenix-heartbeat

Rotation:
  daily

Retention:
  7 copies

Compression:
  gzip with one-cycle delay

Active file:
  root:root 0644
```

## Security Lessons

- Do not make `/var/log` broadly writable to fix an application-specific issue.
- Use dedicated logrotate rules for custom logs.
- Define the correct execution identity with `su` when required.
- Log files may contain operational and security-sensitive information.
- Passwords, tokens, private keys, and secrets must never be written to logs.
- Authentication logs should not be broadly readable.
- Rotation is not a replacement for backup or centralized logging.

## Troubleshooting Lessons

- Always validate a new rule in debug mode first.
- A valid configuration may still skip rotation because the interval has not passed.
- `notifempty` prevents empty logs from rotating.
- `missingok` suppresses errors for missing logs.
- Incorrect ownership can prevent an application from reopening a new log.
- Some applications require a reload or signal after rotation.
- `copytruncate` should only be used when the application cannot reopen its log.
- Forced rotation should be used carefully because repeated tests consume retention slots.

## Portfolio Evidence

This lab demonstrates practical experience with:

- Linux log directory structure;
- traditional log files;
- persistent systemd journal storage;
- file metadata and inode analysis;
- custom logrotate configuration;
- syntax and security validation;
- rotation retention;
- gzip compression;
- delayed compression;
- systemd timer inspection;
- oneshot service behavior;
- state-file interpretation;
- post-rotation application verification;
- logging security and recovery considerations.

## Related Runbook

```text
runbooks/logging-and-logrotate-lab.md
```
---

# Scheduled Tasks and Automation

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Scheduled Tasks and Automation |
| Subtopic | cron, systemd Timers, Troubleshooting, and Safe Recurring Jobs |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A practical comparison between traditional `cron` scheduling and `systemd` timers was completed on `phoenix-linux-01`.

The lab covered:

- listing and interpreting systemd timers;
- understanding timer-to-service relationships;
- reviewing a custom oneshot service;
- inspecting the traditional cron daemon;
- reading system-wide cron syntax;
- creating a user-level cron job;
- defining an explicit cron environment;
- verifying execution through logs;
- testing a controlled failure;
- restoring the valid job;
- removing the temporary recurring schedule safely.

## systemd Timer Model

The existing Project Phoenix automation consists of:

```text
phoenix-heartbeat.timer
phoenix-heartbeat.service
```

The timer defines when the job runs:

```ini
[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Unit=phoenix-heartbeat.service
```

The service defines what runs:

```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/phoenix-heartbeat.sh
```

The operating model is:

```text
timer → service → script
```

## Enabled State

The final unit state was:

```text
phoenix-heartbeat.timer   → enabled
phoenix-heartbeat.service → disabled
```

This is correct for a oneshot service activated only by its timer.

## Cron Service

The traditional cron daemon was confirmed as:

```text
Loaded: loaded
Enabled: yes
Active: active (running)
```

Cron activity was visible in the system logs.

## System-Wide Cron Syntax

The main system crontab is:

```text
/etc/crontab
```

Its format is:

```text
minute hour day-of-month month day-of-week user command
```

Example:

```cron
17 * * * * root cd / && run-parts --report /etc/cron.hourly
```

System-wide cron files include an explicit user field.

User crontabs do not.

## User-Level Cron Test

The user `igor` initially had no personal crontab.

A temporary test job was created:

```cron
*/2 * * * * /usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1
```

The job ran every two minutes and produced timestamps such as:

```text
2026-07-22T19:48:01+00:00
2026-07-22T19:50:01+00:00
2026-07-22T19:52:01+00:00
```

## Cron Environment

The user crontab was configured explicitly:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin
```

This documented the actual non-interactive environment used by cron.

The user's interactive Fish shell does not automatically apply to scheduled cron jobs.

## Execution Audit

The cron event was confirmed in:

```text
/var/log/syslog
```

Observed entry:

```text
(igor) CMD (/usr/bin/date --iso-8601=seconds >> /mnt/phoenix-data/cron-lab.log 2>&1)
```

This verified:

```text
Scheduler: cron
Execution user: igor
Command: exact configured command
Audit location: /var/log/syslog
```

## Controlled Failure Test

The valid command was temporarily replaced with:

```cron
*/2 * * * * /usr/bin/nonexistent-phoenix-command >> /mnt/phoenix-data/cron-error.log 2>&1
```

The captured error was:

```text
/bin/sh: 1: /usr/bin/nonexistent-phoenix-command: not found
```

This confirmed that:

- cron executed the job;
- `/bin/sh` handled the command;
- the failure was intentional;
- `stderr` was redirected correctly;
- the error was available for troubleshooting.

## Recovery

The original `date` command was restored.

New timestamps appeared after recovery:

```text
2026-07-22T19:54:01+00:00
2026-07-22T19:56:01+00:00
```

This proved that the valid job resumed correctly.

The temporary error log was then removed.

## Cleanup

The recurring test line was removed from the user crontab.

Only the explicit environment remained:

```cron
SHELL=/bin/sh
PATH=/usr/local/bin:/usr/bin:/bin
```

The final `cron-lab.log` metadata was:

```text
Modify: 2026-07-22 19:56:01 UTC
Size: 130 bytes
```

No new entries appeared after the schedule was removed.

## Cron and systemd Timer Comparison

| Feature | cron | systemd timer |
|---|---|---|
| Simple recurring jobs | Strong | Strong |
| User-level scheduling | Simple | More configuration |
| Structured status | Limited | Native |
| Logging | syslog and redirection | systemd journal |
| Dependencies | Limited | Native |
| Missed-run recovery | Limited | `Persistent=true` |
| Boot-relative scheduling | Limited | Native |
| Randomized delay | Limited | Native |
| Sandboxing | Limited | Strong |
| Failure state | Indirect | Unit status |

## When to Use cron

Cron is appropriate for:

- simple recurring commands;
- personal user jobs;
- lightweight scripts;
- existing legacy automation;
- schedules that do not require dependencies or complex service controls.

## When to Use systemd Timers

Systemd timers are appropriate when the task requires:

- service dependencies;
- structured journal logging;
- failure status;
- boot-relative schedules;
- resource controls;
- a dedicated service user;
- sandboxing;
- missed-run handling;
- integration with other systemd units.

## Security Lessons

- Do not store passwords, tokens, or private keys in crontab lines.
- Use the least privileged execution identity.
- Use full command paths.
- Protect scheduled scripts from unauthorized modification.
- Avoid running every recurring task as `root`.
- Redirect or otherwise capture errors.
- Remove obsolete scheduled jobs.
- Review user and system crontabs regularly.
- Prefer systemd service hardening for sensitive recurring tasks.

## Troubleshooting Lessons

- Verify the installed schedule with `crontab -l`.
- Check `cron.service` status.
- Inspect `/var/log/syslog` for `CRON` entries.
- Confirm system time and timezone.
- Use absolute paths.
- Define required environment values explicitly.
- Confirm directory write permissions.
- Test commands manually before scheduling them.
- Test controlled failure behavior.
- Verify recovery after restoring a valid command.
- Clean up temporary lab jobs.

## Portfolio Evidence

This lab demonstrates practical experience with:

- systemd timers;
- oneshot services;
- timer and service separation;
- cron daemon inspection;
- system-wide crontab syntax;
- user crontab management;
- scheduled-job environment design;
- output and error redirection;
- syslog-based audit verification;
- controlled failure testing;
- recovery validation;
- safe automation cleanup.

## Related Runbook

```text
runbooks/scheduled-tasks-and-automation-lab.md
```
---

# Backup and Restore Fundamentals

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Backup and Restore Fundamentals |
| Subtopic | Archive Creation, Integrity Verification, Isolated Restore, and Recovery |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A complete backup and restore workflow was implemented and tested on `phoenix-linux-01`.

The lab used dedicated test data and covered:

- compressed archive creation;
- archive inspection;
- gzip integrity testing;
- SHA-256 checksum generation and verification;
- simulated source loss;
- isolated restore testing;
- metadata and content verification;
- byte-for-byte comparison;
- restoration to the original location;
- cleanup of temporary restore data.

## Lab Paths

```text
Source:
  /home/igor/phoenix-backup-lab

Backup destination:
  /mnt/phoenix-data/backups

Restore test location:
  /home/igor/phoenix-restore-test

Archive:
  /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz

Checksum:
  /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz.sha256
```

## Source Data

The lab dataset contained:

```text
phoenix-backup-lab/
├── README.md
└── config/
    └── app.conf
```

Configuration content:

```text
service_name=phoenix-demo
backup_enabled=true
```

Documentation content:

```text
# Phoenix Backup Lab

This file is used to verify backup and restore integrity.
```

## Archive Creation

The compressed archive was created with:

```bash
tar -czvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  -C /home/igor \
  phoenix-backup-lab
```

Using:

```text
-C /home/igor
```

ensured that the archive contained a clean relative path:

```text
phoenix-backup-lab/
```

rather than an absolute filesystem path.

## Archive Inspection

The archive was inspected without extraction:

```bash
tar -tzvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

The listing confirmed:

- complete directory structure;
- owner and group metadata;
- permissions;
- file sizes;
- timestamps.

## Compression Integrity

The gzip stream was verified with:

```bash
gzip -t /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

No output indicated a successful integrity test.

This proved that the compressed stream was readable but did not replace a restore test.

## SHA-256 Checksum

A checksum was generated:

```bash
sha256sum /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  > /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz.sha256
```

Recorded value:

```text
bd369e41fbe1112a9d79e4e0d4cf8c9dec4ae6a14c05c477299ddcbc885ca282
```

The checksum was verified with:

```bash
cd /mnt/phoenix-data/backups
sha256sum -c phoenix-backup-lab.tar.gz.sha256
```

Result:

```text
/mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz: OK
```

## Checksum Meaning

A checksum is a digital fingerprint of a file.

It can detect:

- corruption;
- unintended modification;
- incomplete transfer;
- storage errors.

A checksum does not prove:

- that all required files were included;
- that the backup is logically complete;
- that a restore will work;
- that the dependent application can use the restored data.

## Simulated Data Loss

The source directory was intentionally removed:

```bash
rm -rf /home/igor/phoenix-backup-lab
```

Its absence was verified before restore.

The backup archive remained intact on the dedicated lab disk.

## Isolated Restore

The archive was restored first into:

```text
/home/igor/phoenix-restore-test
```

Command:

```bash
tar -xzvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  -C /home/igor/phoenix-restore-test
```

This prevented accidental overwriting of the original path before verification.

## Why Isolated Restore Matters

A safer restore workflow is:

```text
verify checksum
        ↓
inspect archive
        ↓
restore to isolated location
        ↓
verify content and metadata
        ↓
preserve current live data
        ↓
restore to production
        ↓
test the dependent service
        ↓
retain rollback capability
```

This reduces the risk of:

- overwriting valid data;
- restoring the wrong version;
- applying incorrect ownership or permissions;
- introducing unexpected archive contents;
- disrupting a running service.

## Restored Metadata

The isolated restore preserved:

```text
Owner: igor
Group: igor
Directory mode: 775
File mode: 664
Original timestamps: preserved
Directory hierarchy: complete
```

## Content Verification

The restored configuration contained:

```text
service_name=phoenix-demo
backup_enabled=true
```

The restored README contained:

```text
# Phoenix Backup Lab

This file is used to verify backup and restore integrity.
```

## Direct Archive Inspection

A file was read directly from the archive:

```bash
tar -xOzf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  phoenix-backup-lab/config/app.conf
```

The uppercase `-O` option sent the file content to standard output without writing it to disk.

## Byte-for-Byte Comparison

The archive content was compared with the restored file:

```bash
diff \
  <(tar -xOzf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz phoenix-backup-lab/config/app.conf) \
  /home/igor/phoenix-restore-test/phoenix-backup-lab/config/app.conf
```

No output confirmed identical content.

## Restoration to the Original Location

After successful isolated verification, the restored data was copied back:

```bash
cp -a /home/igor/phoenix-restore-test/phoenix-backup-lab \
      /home/igor/
```

The `-a` option preserved:

- structure;
- permissions;
- timestamps;
- symbolic links;
- ownership where permitted.

The original location was successfully restored:

```text
/home/igor/phoenix-backup-lab
```

## Final Cleanup

The temporary isolated restore copy was removed.

Final state:

```text
Original source:
  restored

Verified backup archive:
  retained

SHA-256 checksum:
  retained

Temporary restore copy:
  removed
```

## Backup Validation Layers

```text
1. Backup file exists
2. Archive table of contents is readable
3. Gzip integrity test passes
4. Checksum verification passes
5. Important file content is inspected
6. Isolated restore succeeds
7. Metadata is verified
8. Restored content matches the archive
9. Application-level recovery is tested
```

A backup should not be considered fully trusted until restore behavior has been tested.

## Backup vs Snapshot

A snapshot is useful for:

- fast rollback;
- point-in-time VM or filesystem state;
- short-term protection before changes.

A backup should:

- exist independently from the source;
- survive source deletion or failure;
- support restoration elsewhere;
- have defined retention;
- be regularly verified.

Snapshots are useful, but they should not be the only recovery mechanism.

## 3-2-1 Principle

A stronger backup strategy follows:

```text
3 copies of data
2 different storage media
1 copy off-site
```

This lab produced one local backup on a separate filesystem.

It does not yet provide protection against:

- complete host failure;
- loss of both VM disks;
- storage-controller failure;
- theft;
- fire;
- ransomware affecting the same system.

## Security Lessons

- Do not blindly extract untrusted archives as `root`.
- Inspect archive paths before extraction.
- Avoid unexpected absolute paths.
- Check symbolic links.
- Verify restored ownership and permissions.
- Protect checksum files from modification.
- Encrypt backups containing sensitive information.
- Store important backups separately from the source.
- Stop dependent services before replacing live application data.
- Preserve a rollback copy before production restore.

## GUI-First Workflow

VS Code is useful for:

- creating and reviewing source files;
- inspecting restored directory structures;
- reading restored content;
- visual text comparison;
- documenting results.

The terminal is more reliable for:

- archive creation;
- metadata inspection;
- gzip verification;
- checksums;
- controlled extraction;
- permission preservation;
- byte-for-byte comparison.

Recommended model:

```text
VS Code Explorer
        ↓
Create and inspect files

VS Code integrated terminal
        ↓
Archive, verify, checksum, and restore

VS Code Diff
        ↓
Visual content comparison
```

## Troubleshooting Lessons

- A backup file existing does not prove it is usable.
- An empty command output can mean success for tools such as `gzip -t` and `diff`.
- Archive listing output should not be pasted back into the shell.
- Checksum failure must be investigated before restore.
- Permission errors should not automatically be solved with `sudo`.
- Incorrect ownership can break an application after restore.
- Successful extraction does not prove application recovery.
- Restore validation should include content, metadata, and service behavior.

## Portfolio Evidence

This lab demonstrates practical experience with:

- `tar` archive creation;
- gzip compression;
- archive inspection;
- relative-path backup design;
- SHA-256 checksums;
- integrity verification;
- isolated restore workflows;
- ownership and permission validation;
- timestamp preservation;
- direct archive content inspection;
- process substitution;
- `diff` comparison;
- archive-mode copy;
- recovery planning;
- backup security;
- snapshot and backup distinctions.

## Related Runbook

```text
runbooks/backup-and-restore-fundamentals.md
```
---

# Filesystem Monitoring and Capacity Management

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Filesystem Monitoring and Capacity Management |
| Subtopic | Disk Usage, Inode Usage, Growth Analysis, and Safe Cleanup Planning |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A complete filesystem-capacity investigation was performed on `phoenix-linux-01`.

The lab covered:

- disk usage;
- inode usage;
- top-level directory analysis;
- `/var` and `/home` growth investigation;
- application-aware cleanup;
- VS Code Remote Server version analysis;
- targeted stale-version removal;
- reclaimed-space verification;
- inode-consumption testing with many small files;
- cleanup and baseline restoration.

## Filesystem Baseline

### Root Filesystem

```text
Filesystem: /dev/mapper/ubuntu--vg-ubuntu--lv
Mount point: /
Size: 30G
Used before cleanup: 7.6G
Available before cleanup: 21G
Usage before cleanup: 27%
```

Inode baseline:

```text
Total inodes: 1,966,080
Used: 110,084
Free: 1,855,996
Usage: 6%
```

### Lab Filesystem

```text
Filesystem: /dev/sdb1
Mount point: /mnt/phoenix-data
Size: 3.9G
Used: 44K
Available: 3.7G
Usage: 1%
```

Inode baseline:

```text
Total: 262,144
Used: 18
Free: 262,126
Usage: 1%
```

## Disk Space and Inodes

Disk blocks and inodes are separate filesystem resources.

```text
df -h
```

checks capacity.

```text
df -i
```

checks inode consumption.

A filesystem may report:

```text
No space left on device
```

even when free capacity remains, if all inodes are exhausted.

## Root Usage Analysis

The root filesystem was analyzed with:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Largest areas:

```text
/usr  → 2.3G
/home → 1.9G
/var  → 442M
/etc  → 6.2M
```

The `-x` option prevented the analysis from crossing into the separate `/mnt/phoenix-data` filesystem.

## `/var` Analysis

Observed usage:

```text
/var/lib   → 231M
/var/cache → 146M
/var/log   → 64M
```

### `/var/lib`

Contained important package-management state:

```text
/var/lib/apt  → 189M
/var/lib/dpkg → 29M
```

These directories were identified as critical state and were not modified.

### `/var/cache`

The largest cache area was:

```text
/var/cache/apt → 108M
```

Further inspection showed:

```text
pkgcache.bin    → 54M
srcpkgcache.bin → 54M
```

The downloaded package archive contained:

```text
0 .deb files
```

Therefore, `apt clean` would not have reclaimed meaningful space.

The cache was not removed because the root filesystem already had approximately `21G` free.

## Operational Decision

```text
A cleanup candidate is not automatically a cleanup requirement.
```

Cleanup should be based on:

- actual capacity pressure;
- measured recovery value;
- dependency risk;
- reproducibility;
- operational need.

## `/home` Analysis

The dominant user-space consumer was:

```text
/home/igor/.vscode-server → 1.9G
```

Detailed analysis showed:

```text
/home/igor/.vscode-server/cli/servers → 1.8G
```

Three VS Code Remote Server versions existed:

```text
1.129.0 → 598M
1.129.1 → 598M
1.130.0 → 598M
```

## Active Dependency Verification

The active VS Code Remote process used:

```text
1.129.1
```

The newest installed version was:

```text
1.130.0
```

The stale version was:

```text
1.129.0
```

The cleanup target was verified through:

- active process inspection;
- launcher filenames;
- installation dates;
- server directory structure;
- exact `product.json` version parsing;
- direct size measurement.

## Targeted Cleanup

Removed objects:

```text
Stable-125df4672b8a6a34975303c6b0baa124e560a4f7
code-125df4672b8a6a34975303c6b0baa124e560a4f7
```

Recovered capacity:

```text
Server directory: 598M
Launcher: 32M
Total: approximately 630M
```

The active version and newest version were retained.

## Post-Cleanup Result

```text
VS Code Remote usage:
1.8G → 1.2G
```

```text
Root filesystem used:
7.6G → 7.0G
```

```text
Root usage:
27% → 25%
```

```text
Available capacity:
21G → 22G
```

## Inode Lab

A dedicated test directory was created on:

```text
/mnt/phoenix-data
```

One thousand empty files were created.

Before:

```text
Disk used: 48K
Inodes used: 19
```

After:

```text
Disk used: 68K
Inodes used: 1019
```

Measured result:

```text
1,000 files
approximately 20K additional capacity
1,000 additional inodes
```

This demonstrated that many tiny files can consume inodes without using much disk space.

## Inode Cleanup

The dedicated lab directory was removed.

Final state:

```text
Disk used: 44K
Inodes used: 18
```

The filesystem returned to its original baseline.

## Safe Cleanup Workflow

```text
measure
    ↓
identify
    ↓
inspect
    ↓
verify dependencies
    ↓
select one confirmed target
    ↓
remove
    ↓
measure again
```

## Security Lessons

- Do not run broad `rm -rf` commands against system directories.
- Do not manually remove package state from `/var/lib`.
- Do not delete active runtime versions.
- Verify active processes before removing application data.
- Prefer application-aware cleanup mechanisms.
- Measure reclaimed capacity after every cleanup.
- Preserve logs and data required for troubleshooting or recovery.
- Use least privilege.
- Document every meaningful deletion.

## Troubleshooting Lessons

- Check both `df -h` and `df -i`.
- Use `du -x` to avoid crossing filesystem boundaries.
- Large directories by bytes are not always large inode consumers.
- Modification dates alone are not enough to identify stale data.
- Structured metadata is more reliable than broad text searches.
- A process may keep deleted files open and prevent immediate space recovery.
- Application cache, state, logs, and runtime data must be treated differently.
- Cleanup without evidence can create a larger operational problem than the capacity issue.

## Monitoring Considerations

Recommended metrics:

```text
filesystem used percentage
filesystem available bytes
inode used percentage
inode availability
filesystem growth rate
mount availability
read-only filesystem state
```

Example thresholds:

```text
Warning: 80%
Critical: 90%
```

Thresholds should be adjusted to the filesystem size, growth rate, service importance, and operational response time.

## Portfolio Evidence

This lab demonstrates practical experience with:

- `df`;
- `du`;
- inode analysis;
- filesystem boundary control;
- package cache investigation;
- critical state identification;
- VS Code Remote Server analysis;
- process correlation;
- JSON metadata parsing;
- safe targeted cleanup;
- capacity measurement;
- inode exhaustion simulation;
- cleanup verification;
- operational decision-making.

## Related Runbook

```text
runbooks/filesystem-monitoring-and-capacity-management.md
```
---

# Process Limits and Resource Control

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Process Limits and Resource Control |
| Subtopic | CPU Priority, Shell Limits, systemd Constraints, Verification, and Cleanup |
| System | `phoenix-linux-01` |
| Status | Completed and verified |

## Lab Summary

A complete process and systemd resource-control lab was performed on `phoenix-linux-01`.

The lab covered:

- CPU and memory baselining;
- process inspection;
- process priority with `nice`;
- safe workload termination with `timeout`;
- shell limits through `ulimit`;
- soft and hard file-descriptor limits;
- service-level limits through systemd;
- CPU quota enforcement;
- memory and file-descriptor constraints;
- exit-status handling;
- cleanup of a temporary systemd unit.

## Initial Resource Baseline

Observed system state:

```text
Load average:
  0.00, 0.00, 0.00

Memory:
  Total: 3.8 GiB
  Used: 468 MiB
  Available: 3.3 GiB

Swap:
  Total: 3.1 GiB
  Used: 0 B

Processes:
  112
```

The system had no significant CPU or memory pressure.

## Process Inspection

Processes were sorted by CPU and memory usage using `ps`.

Important fields included:

```text
PID
USER
NI
COMMAND
%CPU
%MEM
```

The `NI` field represents the process nice value.

## Nice Values

Typical range:

```text
-20 → highest scheduling priority
  0 → normal priority
 19 → lowest scheduling priority
```

A test workload was started with:

```bash
nice -n 10 timeout 30s sha256sum /dev/zero >/dev/null &
```

Observed result:

```text
Nice value: 10
CPU usage: approximately 99.8%
```

This proved that `nice` changes relative scheduling priority but does not create a fixed CPU ceiling.

Because no competing workload required the CPU, the low-priority process could still use almost one complete CPU core.

## Timeout Behavior

The workload ended with:

```text
Exit 124
```

Exit status `124` means that the `timeout` utility stopped the command because the configured duration expired.

The process was then verified as no longer running.

## Shell Resource Limits

Current shell limits were inspected with:

```bash
ulimit -a
```

Important observed values:

```text
Open files: 1024
Maximum user processes: 15144
Stack size: 8192 KB
Core file size: 0
CPU time: unlimited
Virtual memory: unlimited
```

## Soft and Hard Limits

The open-file limits were:

```text
Soft limit: 1024
Hard limit: 1048576
```

The soft limit is the active limit.

The hard limit is the maximum value to which the user can normally raise the soft limit.

The current shell soft limit was temporarily raised:

```bash
ulimit -Sn 4096
```

Result:

```text
4096
```

This change applied only to the current shell session and its child processes.

## Shell and systemd Scope

Shell limits apply to:

```text
current shell
child processes launched from that shell
```

Systemd limits apply directly to:

```text
managed service
service processes
service cgroup
```

## Existing Service Limits

The Project Phoenix heartbeat service showed:

```text
CPUQuotaPerSecUSec=infinity
MemoryMax=infinity
LimitNOFILE=524288
LimitNPROC=15144
```

This confirmed that systemd services can have different defaults from interactive shell sessions.

## Temporary Resource-Control Unit

A dedicated lab service was created:

```text
/etc/systemd/system/phoenix-resource-lab.service
```

Initial service configuration:

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

The unit was validated with:

```bash
sudo systemd-analyze verify /etc/systemd/system/phoenix-resource-lab.service
```

No syntax errors were reported.

## File-Descriptor Limit Test

The service journal showed:

```text
256
resource lab completed
```

This proved that:

```ini
LimitNOFILE=256
```

was applied inside the service.

## Applied systemd Values

Systemd reported:

```text
CPUQuotaPerSecUSec=250ms
MemoryMax=67108864
LimitNOFILE=256
```

Interpretation:

```text
250ms CPU time per second = 25% of one CPU core
67108864 bytes = 64 MiB
```

## CPU Quota Test

The service command was changed to:

```ini
ExecStart=/usr/bin/timeout 20s /usr/bin/sha256sum /dev/zero
```

The resource controls remained:

```ini
LimitNOFILE=256
MemoryMax=64M
CPUQuota=25%
```

During execution, the workload showed:

```text
sha256sum → approximately 25.1% CPU
```

This confirmed that:

```ini
CPUQuota=25%
```

was actively enforced.

## Nice Versus CPUQuota

```text
nice
```

changes scheduler priority.

```text
CPUQuota
```

creates an actual CPU consumption ceiling.

Comparison from the lab:

```text
nice test:
  approximately 99.8% CPU

systemd CPUQuota test:
  approximately 25.1% CPU
```

## CPU Accounting

The 20-second test consumed:

```text
5.025 seconds of CPU time
```

Expected calculation:

```text
20 seconds × 25% = 5 seconds
```

The measured CPU time closely matched the configured quota.

## Expected Exit Status

The first CPU quota test was marked failed because `timeout` returned:

```text
124
```

The service was updated with:

```ini
SuccessExitStatus=124
```

Final unit configuration:

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

After reloading and retesting, the service completed with:

```text
Deactivated successfully
Finished
Consumed 5.025s CPU time
```

The expected timeout result was no longer treated as a service failure.

## Cleanup

The temporary service file was removed:

```bash
sudo rm /etc/systemd/system/phoenix-resource-lab.service
sudo systemctl daemon-reload
```

Verification returned:

```text
Unit phoenix-resource-lab.service could not be found.
```

No temporary lab service remained installed.

## Important Resource Controls

### File descriptors

```ini
LimitNOFILE=4096
```

Useful for services handling many files, sockets, or network connections.

### Memory ceiling

```ini
MemoryMax=512M
```

Limits the memory used by the service cgroup.

### CPU ceiling

```ini
CPUQuota=50%
```

Limits the service to approximately half of one CPU core.

### Relative CPU weight

```ini
CPUWeight=100
```

Controls relative CPU importance during contention but does not create a fixed maximum.

### Process and thread control

```ini
LimitNPROC=512
TasksMax=512
```

Can help contain runaway process or thread creation.

## Security Lessons

Resource limits can reduce the impact of:

- runaway CPU workloads;
- memory leaks;
- descriptor leaks;
- fork bombs;
- compromised services;
- poorly configured applications.

Incorrectly low limits can also create denial-of-service conditions against legitimate workloads.

Limits should therefore be:

- based on measured behavior;
- applied per service;
- tested under load;
- monitored after deployment;
- documented with rollback steps.

## Troubleshooting Lessons

- `nice` does not impose a fixed CPU percentage.
- Exit code `124` is expected when `timeout` expires.
- Shell limits and systemd service limits are separate.
- `daemon-reload` is required after changing unit files.
- `systemctl show` reveals normalized applied values.
- CPU accounting can validate quota enforcement.
- `SuccessExitStatus` should accept only understood and intentional exit codes.
- A oneshot service being inactive after completion is normal.
- Temporary units should be removed after testing.

## Portfolio Evidence

This lab demonstrates practical experience with:

- CPU and memory baselining;
- process inspection;
- Linux scheduling priority;
- `nice`;
- `timeout`;
- exit-status interpretation;
- `ulimit`;
- soft and hard limits;
- file descriptors;
- systemd resource controls;
- cgroup CPU quotas;
- memory ceilings;
- systemd accounting;
- service verification;
- controlled cleanup.

## Related Runbook

```text
runbooks/process-limits-and-resource-control.md
```