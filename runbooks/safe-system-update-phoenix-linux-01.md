# Safe System Update — phoenix-linux-01

## Document Information

| Field | Value |
|---|---|
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Hypervisor | Proxmox VE |
| Management IP | `192.168.20.101` |
| Update method | APT |
| Status | Completed and verified |

## Purpose

This runbook documents a controlled operating system update for the `phoenix-linux-01` lab virtual machine.

The objective was to apply available package updates while maintaining a safe recovery path and verifying system availability after reboot.

## Change Safety Controls

The following controls were used before the update:

- The change was performed on a dedicated lab virtual machine.
- A Proxmox snapshot was created before package installation.
- Available disk space and system load were checked.
- The package upgrade was simulated before execution.
- The simulation confirmed that no packages would be removed.
- SSH key-based access had already been verified.
- Proxmox console access remained available as a recovery method.

## Pre-Update Snapshot

A Proxmox snapshot was created with the following name:

```text
before-system-update
```

Description:

```text
Clean checkpoint before the first Project Phoenix system update.
```

The snapshot included the VM state and disk state.

For future routine update snapshots, saving RAM state is usually unnecessary unless preservation of the exact running state is required.

## Pre-Update System Check

The following commands were used:

```bash
uptime
df -h /
```

Verified results:

- system load was effectively zero;
- the system had been stable before the change;
- the root filesystem used approximately 49%;
- approximately 7.2 GB remained available.

This confirmed that the system had sufficient storage and no immediate load-related risk.

## Repository Metadata Refresh

The package metadata was refreshed:

```bash
sudo apt update
```

Result:

```text
111 packages can be upgraded.
```

The update repositories were reachable and package indexes were downloaded successfully.

## Review of Available Updates

The available updates were inspected with:

```bash
apt list --upgradable
```

The update set included core components such as:

- APT;
- AppArmor;
- cloud-init;
- Netplan;
- nftables;
- rsyslog;
- snapd;
- LVM tools;
- initramfs tools;
- OpenSSH-related dependencies;
- system utilities and libraries.

## Upgrade Simulation

Before installing packages, the upgrade was simulated:

```bash
sudo apt upgrade --simulate
```

The summary was extracted with:

```bash
sudo apt upgrade --simulate | grep -E 'upgraded,|REMOVED|kept back'
```

Confirmed result:

```text
111 upgraded, 8 newly installed, 0 to remove and 0 not upgraded.
```

This confirmed that:

- 111 packages would be upgraded;
- 8 required packages would be added;
- no packages would be removed;
- no packages would remain held back.

## Package Upgrade

The upgrade was started with:

```bash
sudo apt upgrade
```

The installation completed successfully.

The update process restarted several services, including:

```text
cron
packagekit
polkit
ssh
systemd-journald
systemd-networkd
systemd-resolved
systemd-timesyncd
systemd-udevd
udisks2
upower
```

Some service restarts and user-session updates were deferred until reboot.

## Reboot Requirement Check

The system reported:

```text
System restart required
```

The restart requirement was also checked using:

```bash
test -f /var/run/reboot-required \
  && cat /var/run/reboot-required \
  || echo "No reboot required"
```

A controlled reboot was therefore scheduled.

## Controlled Reboot

The system was rebooted with:

```bash
sudo reboot
```

The SSH connection closed as expected.

After the VM returned online, a new SSH connection was opened from the administration laptop:

```bash
ssh phoenix-linux-01
```

The connection succeeded using the existing SSH alias and Ed25519 key.

## Post-Update Verification

The following checks were performed after reboot:

```bash
uptime
test -f /var/run/reboot-required \
  && echo "Reboot still required" \
  || echo "No reboot required"
```

Verified results:

- the system had recently rebooted;
- SSH access was operational;
- network connectivity was restored;
- key-based authentication remained functional;
- no additional reboot was required.

## Final Result

| Check | Result |
|---|---|
| Proxmox snapshot created | Passed |
| Package metadata refreshed | Passed |
| Upgrade simulated | Passed |
| Packages removed | None |
| Packages held back | None |
| Package upgrade completed | Passed |
| Controlled reboot completed | Passed |
| Network restored | Passed |
| SSH restored | Passed |
| Key authentication preserved | Passed |
| Additional reboot required | No |

## Recovery Procedure

If the VM fails after an update:

1. Open the VM console in Proxmox.
2. Verify whether the operating system is booting.
3. Check the VM network interface and IP address.
4. Review boot and package logs.
5. Attempt normal package repair only if the system is accessible.
6. If recovery is not practical, shut down the VM.
7. Restore the `before-system-update` snapshot.
8. Start the VM and verify SSH, networking, and system health.
9. Investigate the failed update before attempting it again.

Useful diagnostic commands include:

```bash
systemctl --failed
journalctl -p err -b
```

## Snapshot Cleanup

The pre-update snapshot should not be retained indefinitely.

After the system has remained stable and the update has been fully verified, the snapshot may be removed from Proxmox to prevent unnecessary storage consumption.

Snapshot deletion should only occur after confirming:

- the VM boots reliably;
- SSH access works;
- networking is stable;
- no important services have failed;
- no rollback is expected.

## Lessons Learned

- A package update should be treated as a controlled infrastructure change.
- Package metadata refresh and package installation are separate operations.
- Upgrade simulation provides visibility before system modification.
- The number of upgraded packages alone does not determine risk.
- Package removals and held packages require special attention.
- A snapshot provides a recovery point but does not replace a full backup.
- Active services may continue using old binaries until reboot.
- SSH and network functionality must be tested after system restart.
- A successful command does not complete the change; post-change verification does.
- Temporary snapshots should be removed after the rollback window expires.

## Outcome

The first controlled operating system update of `phoenix-linux-01` was completed successfully.

The lab demonstrated:

- pre-change risk assessment;
- snapshot-based rollback preparation;
- package update inspection;
- upgrade simulation;
- controlled package installation;
- reboot management;
- post-update service verification;
- recovery planning;
- professional infrastructure documentation.