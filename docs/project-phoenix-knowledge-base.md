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