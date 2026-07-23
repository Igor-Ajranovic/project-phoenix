# Package Management and Repository Safety

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Package Management and Repository Safety |
| Subtopic | Safe Updates, Upgrade Simulation, Cleanup, and Verification |
| System | `phoenix-linux-01` |
| Operating System | Ubuntu Server 24.04 LTS |
| Status | Completed |

## Purpose

This runbook documents a safe and repeatable package-management workflow for an Ubuntu Server system.

The objective is to:

- refresh package metadata;
- review available upgrades;
- install updates safely;
- simulate potentially destructive operations;
- remove verified orphaned dependencies;
- confirm package and service health afterward;
- preserve an audit trail of package changes.

---

## Core APT Commands

### Refresh Package Metadata

```bash
sudo apt update
```

This command downloads current package metadata from configured repositories.

It does not install package upgrades.

### Review Available Upgrades

```bash
apt list --upgradable
```

This displays packages that have newer versions available.

### Install Standard Upgrades

```bash
sudo apt upgrade
```

This installs available upgrades without removing installed packages to resolve dependency changes.

### Simulate an Upgrade

```bash
sudo apt-get --simulate upgrade
```

A simulation shows the planned actions without modifying the system.

### Simulate a Full Upgrade

```bash
sudo apt-get --simulate full-upgrade
```

A full upgrade may install or remove packages when required to resolve dependencies.

Simulation should be used before approving unexpected changes.

---

## Repository Baseline

The active repositories were confirmed as official Ubuntu sources:

```text
http://rs.archive.ubuntu.com/ubuntu/
http://security.ubuntu.com/ubuntu/
```

Enabled suites included:

```text
noble
noble-updates
noble-backports
noble-security
```

Enabled components included:

```text
main
restricted
universe
multiverse
```

No active third-party APT repositories were detected.

The file:

```text
/etc/apt/sources.list.d/ubuntu.sources.curtin.orig
```

is an installer backup and is not loaded as an active APT source.

---

## Package Health Baseline

The following checks were used:

```bash
sudo dpkg --audit
```

Checks for packages that are unpacked, partially installed, or not fully configured.

```bash
sudo apt-get check
```

Checks the installed package dependency state.

```bash
apt-mark showhold
```

Displays packages intentionally held at their current version.

Observed result:

```text
No incomplete packages
No broken dependencies
No held packages
```

---

## Automatic Security Updates

The package was confirmed as installed:

```text
unattended-upgrades
```

The service was enabled and active.

The system uses two standard timers:

```text
apt-daily.timer
apt-daily-upgrade.timer
```

Their roles are:

```text
apt-daily.timer
→ refresh package metadata

apt-daily-upgrade.timer
→ run unattended security upgrades
```

The configuration enables daily checks:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Security updates are automatically allowed from:

```text
Ubuntu noble-security
```

Regular updates from `noble-updates` remain under manual administrator control.

---

## Reboot Requirement Check

After package upgrades, verify whether a reboot is required:

```bash
if [ -f /var/run/reboot-required ]; then
  cat /var/run/reboot-required
  cat /var/run/reboot-required.pkgs 2>/dev/null
else
  echo "No reboot required"
fi
```

Observed result:

```text
No reboot required
```

---

## Safe Autoremove Workflow

APT reported two automatically installed packages that were no longer required:

```text
libfwupd2
libgusb2
```

Before removing them, the operation was simulated:

```bash
sudo apt autoremove --simulate
```

The simulation confirmed that only those two packages would be removed.

The currently installed `fwupd` package was verified to depend on:

```text
libfwupd3
libusb-1.0-0
```

It no longer depended on the older libraries.

The cleanup was then performed:

```bash
sudo apt autoremove
```

Removed packages:

```text
libfwupd2
libgusb2
```

---

## Post-Cleanup Verification

After cleanup, the following checks were performed:

```bash
sudo dpkg --audit
sudo apt-get check
systemctl --failed --no-pager
```

Observed result:

```text
No incomplete packages
No dependency errors
0 failed systemd units
```

This confirmed that the package cleanup did not damage the operating system or its services.

---

## Package Audit Trail

APT records high-level package operations in:

```text
/var/log/apt/history.log
```

The cleanup entry recorded:

```text
Commandline: apt autoremove
Requested-By: igor
Remove: libgusb2, libfwupd2
```

Low-level package state changes are recorded in:

```text
/var/log/dpkg.log
```

These logs are useful for determining:

- when a package changed;
- which command initiated the change;
- which user requested it;
- which package version was installed or removed.

---

## Manual and Automatic Packages

APT tracks packages as either manually or automatically installed.

Manual packages:

```text
22
```

Automatic packages:

```text
657
```

Manual packages define the intended system role and are protected from automatic cleanup.

Automatic packages are usually dependencies. They become `autoremove` candidates only when no installed package requires them.

Important manually retained meta-packages included:

```text
linux-generic
ubuntu-minimal
ubuntu-server
ubuntu-server-minimal
ubuntu-standard
openssh-server
qemu-guest-agent
```

---

## Safe Update Procedure

Use this workflow for routine Ubuntu Server maintenance:

```text
1. Refresh package metadata
2. Review available upgrades
3. Simulate unexpected or large operations
4. Apply approved upgrades
5. Review autoremove candidates
6. Simulate autoremove
7. Remove only verified orphaned packages
8. Check package integrity
9. Check failed services
10. Check whether a reboot is required
11. Review APT history when troubleshooting
```

Recommended commands:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
```

Optional cleanup validation:

```bash
sudo apt autoremove --simulate
sudo apt autoremove
```

Final health check:

```bash
sudo dpkg --audit
sudo apt-get check
systemctl --failed --no-pager
```

---

## Security Guidelines

- Use only trusted repositories.
- Review new repository signing keys before installation.
- Avoid copying unknown repository commands from websites.
- Simulate package removals before approving them.
- Do not use `autoremove` blindly on production systems.
- Review packages that affect networking, SSH, storage, boot, or authentication.
- Keep security updates enabled.
- Confirm console access before kernel or bootloader maintenance.
- Check whether a reboot is required after important updates.
- Preserve APT and `dpkg` logs for troubleshooting.

---

## Recovery Considerations

Package operations are not automatically transactional.

Before high-risk changes:

- create a Proxmox snapshot;
- verify console access;
- record the packages being changed;
- preserve relevant configuration files;
- simulate removals;
- keep package logs available.

Useful package database files include:

```text
/var/lib/dpkg/status
/var/lib/dpkg/status-old
/var/backups/dpkg.status.*
```

These files support package-database recovery but do not replace a complete system backup.

---

## Troubleshooting Quick Reference

### Package Metadata Errors

```bash
sudo apt update
```

Review mirror, DNS, network, signature, and repository errors.

### Broken Dependencies

```bash
sudo apt-get check
```

### Incomplete Package Configuration

```bash
sudo dpkg --audit
```

### Available Upgrades

```bash
apt list --upgradable
```

### Held Packages

```bash
apt-mark showhold
```

### Planned Autoremove

```bash
sudo apt autoremove --simulate
```

### Package History

```bash
sudo tail -n 30 /var/log/apt/history.log
```

### Failed Services After Maintenance

```bash
systemctl --failed --no-pager
```

### Reboot Requirement

```bash
test -f /var/run/reboot-required && cat /var/run/reboot-required
```

---

## Outcome

The package-management baseline for `phoenix-linux-01` was verified as healthy.

Final state:

```text
Package metadata: up to date
Available upgrades: none
Broken dependencies: none
Held packages: none
Third-party repositories: none
Failed systemd units: none
Reboot required: no
```

Two obsolete automatic dependencies were safely removed after simulation and dependency verification.

The system retained working automatic security updates and a clear package-operation audit trail.