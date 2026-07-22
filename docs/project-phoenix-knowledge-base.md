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