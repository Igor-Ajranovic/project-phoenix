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