# Boot Process and Recovery Fundamentals

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Boot Process and Recovery Fundamentals |
| Subtopic | Boot Analysis, Journal Review, Kernel Startup, systemd Targets, and Safe Recovery Planning |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Status | Completed and verified |

## Purpose

This runbook documents a practical Linux boot and recovery investigation on the `phoenix-linux-01` lab virtual machine.

The lab covered:

- measuring kernel and userspace startup time;
- identifying slow-starting systemd units;
- distinguishing `systemd-analyze blame` from the critical boot path;
- inspecting the default systemd target;
- reviewing persistent boot history;
- examining current and previous boot errors;
- identifying the active kernel and initramfs;
- reading the kernel command line;
- verifying root filesystem remount behavior;
- inspecting early kernel messages;
- tracing LVM activation;
- comparing rescue and emergency modes;
- validating `/etc/fstab`;
- verifying the active swap file;
- confirming an existing `/etc/fstab` rollback copy;
- reloading systemd after configuration changes;
- building a safe recovery workflow without intentionally breaking boot.

No GRUB configuration, kernel parameters, or boot-critical files were intentionally modified.

---

# Environment

```text
System: phoenix-linux-01
Operating system: Ubuntu Server 24.04.1 LTS
Virtualization: Proxmox VE / KVM / QEMU
Kernel: 6.8.0-136-generic
Root filesystem: /dev/mapper/ubuntu--vg-ubuntu--lv
Root filesystem type: ext4
Boot filesystem: /dev/sda2
Additional persistent filesystem: /dev/sdb1
Additional mount point: /mnt/phoenix-data
```

Recovery access is available through the Proxmox virtual console.

This is important because rescue and emergency modes may not provide working network access or SSH.

---

# Part 1 — Measuring Boot Time

The current boot duration was inspected with:

```bash
systemd-analyze
systemd-analyze time
```

Observed result:

```text
Startup finished in 1.834s (kernel) + 3.876s (userspace) = 5.711s
graphical.target reached after 3.858s in userspace
```

## Interpretation

The total startup time consisted of:

```text
Kernel startup:    1.834 seconds
Userspace startup: 3.876 seconds
Total startup:     5.711 seconds
```

The system reached its default target after:

```text
3.858 seconds of userspace initialization
```

This represents a fast and healthy boot for the lab VM.

---

# Part 2 — Kernel and Userspace Boot Phases

## Kernel Phase

The kernel phase includes:

- loading the Linux kernel image;
- detecting virtual CPU and memory;
- identifying KVM and QEMU hardware;
- initializing drivers;
- reading the kernel command line;
- starting early device discovery;
- preparing access to the root filesystem.

## Userspace Phase

The userspace phase begins when the kernel starts the init system.

On this system, the init system is:

```text
systemd
```

Systemd then:

- mounts filesystems;
- activates swap;
- initializes networking;
- starts system services;
- activates targets;
- starts login and remote-access services;
- reaches the configured default target.

---

# Part 3 — Identifying Slow-Starting Units

Units were sorted by activation duration:

```bash
systemd-analyze blame | head -n 15
```

Observed results included:

```text
13.589s apt-daily-upgrade.service
 1.986s systemd-networkd-wait-online.service
 637ms dev-mapper-ubuntu\x2d\x2dvg\x2dubuntu\x2d\x2dlv.device
 564ms motd-news.service
 417ms apt-daily.service
 398ms fwupd-refresh.service
 301ms apparmor.service
```

## Important Limitation

`systemd-analyze blame` does not prove that a unit delayed the final boot target.

Units often run in parallel.

A unit may report a long activation duration while not being part of the blocking path to the default target.

This was demonstrated by:

```text
apt-daily-upgrade.service → 13.589 seconds
```

while the total boot completed in:

```text
5.711 seconds
```

Therefore:

```text
A long blame duration does not automatically mean a boot bottleneck.
```

---

# Part 4 — Inspecting the Critical Boot Path

The actual critical dependency chain was inspected with:

```bash
systemd-analyze critical-chain
```

Observed result:

```text
graphical.target @3.858s
└─multi-user.target @3.856s
  └─apport.service @3.600s +255ms
    └─remote-fs.target @3.595s
      └─remote-fs-pre.target @3.595s
```

## Interpretation

The critical path to the default target passed through:

```text
remote-fs-pre.target
        ↓
remote-fs.target
        ↓
apport.service
        ↓
multi-user.target
        ↓
graphical.target
```

The largest visible delay on this path was:

```text
apport.service → 255 milliseconds
```

This confirmed that there was no meaningful boot bottleneck.

## Blame Versus Critical Chain

```text
systemd-analyze blame
```

shows unit activation durations.

```text
systemd-analyze critical-chain
```

shows the dependency path that actually determined when the target was reached.

Both are useful, but they answer different questions.

---

# Part 5 — Inspecting the Default Target

The configured default target was checked with:

```bash
systemctl get-default
```

Result:

```text
graphical.target
```

The effective vendor link was inspected with:

```bash
ls -l /usr/lib/systemd/system/default.target
```

Result:

```text
/usr/lib/systemd/system/default.target -> graphical.target
```

No local override existed at:

```text
/etc/systemd/system/default.target
```

## `graphical.target` on a Server

The target definition was inspected:

```bash
systemctl cat graphical.target
```

Important directives:

```ini
Requires=multi-user.target
Wants=display-manager.service
After=multi-user.target rescue.service rescue.target display-manager.service
```

`graphical.target` requires:

```text
multi-user.target
```

but only wants:

```text
display-manager.service
```

The `Wants=` relationship does not require a display manager to exist or start successfully.

Therefore, an Ubuntu Server system can reach `graphical.target` without running a desktop environment.

---

# Part 6 — Persistent Boot History

Available boot sessions were listed with:

```bash
journalctl --list-boots
```

Observed history:

```text
-3 → oldest retained boot
-2 → older boot
-1 → previous boot
 0 → current boot
```

The system retained four boot sessions.

Current boot:

```text
Started: 2026-07-21 20:45:29 UTC
```

Previous boot:

```text
Started: 2026-07-20 21:05:14 UTC
Ended:   2026-07-21 20:45:20 UTC
```

## Persistent Journal

The ability to inspect older boot sessions confirms that the journal is persistent.

Persistent journal storage was previously identified at:

```text
/var/log/journal
```

This is valuable for troubleshooting failures that occurred before a reboot.

---

# Part 7 — Reviewing Current-Boot Warnings

Warnings and more serious messages from the current boot were displayed with:

```bash
journalctl -b 0 -p warning --no-pager
```

Observed messages included:

```text
ACPI MMCONFIG warning
device reset messages
LVM informational messages
cron EXTRA_OPTS warning
D-Bus unknown group "power" warnings
a known Project Phoenix lab service failure
```

## ACPI MMCONFIG Warning

Example:

```text
fail to add MMCONFIG information
```

In this KVM/QEMU virtual environment, the system still booted and operated normally.

The warning did not indicate a current functional failure.

## Virtual Disk Reset Messages

Example:

```text
Power-on or device reset occurred
```

These appeared during virtual disk detection and were consistent with normal VM startup.

## LVM Messages

Example:

```text
PV /dev/sda3 online, VG ubuntu-vg is complete
```

This was a successful LVM activation message, even though it appeared in the warning-filtered output due to the message priority used by the emitting component.

## Cron Environment Warning

Example:

```text
Referenced but unset environment variable evaluates to an empty string: EXTRA_OPTS
```

The cron daemon had already been verified as active and functional.

The unset variable was optional and did not prevent operation.

## D-Bus Group Warning

Example:

```text
Unknown group "power" in message bus configuration file
```

The warning did not prevent the system from reaching its normal target.

It may be investigated separately if it persists or causes a functional issue.

## Known Lab Failure

The current journal contained:

```text
phoenix-resource-lab.service: Failed with result 'exit-code'
```

This was a known controlled test from the previous resource-management lab.

The service initially returned:

```text
status 124
```

because `timeout` intentionally stopped the workload.

The condition was later corrected with:

```ini
SuccessExitStatus=124
```

and the temporary service was removed.

It was not an unresolved system problem.

---

# Part 8 — Reviewing Serious Current-Boot Errors

Only error-level messages were displayed:

```bash
journalctl -b 0 -p err --no-pager
```

Result:

```text
Failed to start phoenix-resource-lab.service
```

This was the known historical lab event.

No other error-level events were present for the current boot.

---

# Part 9 — Reviewing the Previous Boot

The previous boot was checked with:

```bash
journalctl -b -1 -p err --no-pager
```

Result:

```text
-- No entries --
```

This confirmed that the previous boot contained no journal events at the `err` level or higher.

## Operational Conclusion

```text
Current boot:
  one known and resolved lab-generated error

Previous boot:
  no serious errors
```

---

# Part 10 — Current Failed Unit State

The current systemd failure state was checked with:

```bash
systemctl --failed
```

Result:

```text
0 loaded units listed
```

This demonstrated an important distinction.

```text
journalctl
```

stores historical events.

```text
systemctl --failed
```

shows units that are currently in the failed state.

A historical error can remain in the journal after the service has been fixed, reset, or removed.

---

# Part 11 — Identifying the Active Kernel

The currently running kernel was checked with:

```bash
uname -r
```

Result:

```text
6.8.0-136-generic
```

The corresponding boot files were inspected:

```bash
ls -lh /boot/vmlinuz-* /boot/initrd.img-*
```

Observed files:

```text
/boot/vmlinuz-6.8.0-136-generic
/boot/initrd.img-6.8.0-136-generic
```

Sizes:

```text
Kernel image: approximately 15M
Initramfs:     approximately 73M
```

---

# Part 12 — Kernel and Initramfs Roles

## Kernel Image

```text
vmlinuz-6.8.0-136-generic
```

The bootloader loads this compressed Linux kernel image into memory.

The kernel then:

- initializes CPU support;
- detects memory;
- initializes drivers;
- detects storage;
- starts the early userspace process;
- prepares to mount the root filesystem.

## Initial RAM Filesystem

```text
initrd.img-6.8.0-136-generic
```

The initramfs provides a temporary early userspace environment.

It contains tools and drivers required before the real root filesystem is available.

On this VM, initramfs is especially important because the root filesystem resides on LVM:

```text
/dev/mapper/ubuntu--vg-ubuntu--lv
```

The early boot environment must activate the LVM volume group before mounting `/`.

---

# Part 13 — Kernel Command Line

The kernel command line was inspected with:

```bash
cat /proc/cmdline
```

Result:

```text
BOOT_IMAGE=/vmlinuz-6.8.0-136-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro
```

## `BOOT_IMAGE`

```text
BOOT_IMAGE=/vmlinuz-6.8.0-136-generic
```

Identifies the kernel image selected by the bootloader.

## `root=`

```text
root=/dev/mapper/ubuntu--vg-ubuntu--lv
```

Defines the block device that becomes the root filesystem.

## `ro`

```text
ro
```

Requests that the root filesystem initially be mounted read-only during early boot.

This allows early checks and initialization to occur before normal read-write operation begins.

---

# Part 14 — Root Filesystem Remount Behavior

The active root mount was inspected with:

```bash
findmnt /
```

Result:

```text
SOURCE: /dev/mapper/ubuntu--vg-ubuntu--lv
FSTYPE: ext4
OPTIONS: rw,relatime
```

This confirmed the expected transition:

```text
kernel command line specifies ro
        ↓
root is mounted read-only during early boot
        ↓
initial checks and activation complete
        ↓
systemd remounts root read-write
        ↓
normal operation continues with rw
```

## `relatime`

The `relatime` mount option reduces unnecessary disk writes caused by access-time updates.

Linux updates access time selectively rather than after every read operation.

---

# Part 15 — Early Kernel Messages

The beginning of the kernel journal was inspected with:

```bash
journalctl -b 0 -k --no-pager | head -n 25
```

Important observations included:

```text
Linux version 6.8.0-136-generic
Kernel command line detected
CPU vendor support initialized
BIOS-provided memory map detected
NX protection active
QEMU virtual hardware detected
KVM hypervisor detected
KVM clock source selected
```

## Virtualization Detection

The kernel reported:

```text
DMI: QEMU Standard PC
Hypervisor detected: KVM
```

This confirmed that the VM runs on KVM with QEMU-presented virtual hardware.

## NX Protection

The kernel reported:

```text
NX (Execute Disable) protection: active
```

NX protection helps prevent execution of code from memory areas intended to contain data.

It is an important memory-protection feature.

## Memory Map

The BIOS-provided physical RAM map showed:

- usable memory regions;
- reserved memory regions;
- memory exposed by the virtual machine configuration.

---

# Part 16 — LVM Activation During Boot

LVM-related journal events were filtered with:

```bash
journalctl -b 0 --no-pager \
  | grep -E 'ubuntu-vg|ubuntu--lv|LVM|lvm' \
  | head -n 20
```

Observed events:

```text
Listening on lvm2-lvmpolld.socket
Starting lvm2-monitor.service
1 logical volume in volume group ubuntu-vg monitored
PV /dev/sda3 online
VG ubuntu-vg is complete
VG ubuntu-vg finished
```

## LVM Boot Flow

```text
/dev/sda3
    ↓
LVM physical volume detected
    ↓
ubuntu-vg volume group completed
    ↓
ubuntu-lv logical volume activated
    ↓
root filesystem became available
    ↓
system boot continued
```

This verified that the LVM-backed root filesystem was correctly activated during startup.

---

# Part 17 — Rescue Target

The rescue target was inspected with:

```bash
systemctl cat rescue.target
```

Important configuration:

```ini
Requires=sysinit.target rescue.service
After=sysinit.target rescue.service
AllowIsolate=yes
```

## Rescue Mode

Rescue mode starts more of the base system before providing a recovery shell.

It generally includes:

- early system initialization;
- local filesystems required by system initialization;
- basic device management;
- a root recovery shell.

It is suitable for administrative recovery when the system can complete basic initialization.

Possible uses include:

- correcting service configuration;
- fixing permissions;
- restoring configuration files;
- inspecting filesystems;
- disabling a broken service;
- repairing non-critical mount configuration.

---

# Part 18 — Emergency Target

The emergency target was inspected with:

```bash
systemctl cat emergency.target
```

Important configuration:

```ini
Requires=emergency.service
After=emergency.service
AllowIsolate=yes
```

## Emergency Mode

Emergency mode is more minimal than rescue mode.

It provides the smallest practical recovery environment when the normal boot path cannot progress.

Possible uses include:

- broken `/etc/fstab`;
- failed essential mount;
- damaged root filesystem;
- missing critical early-boot device;
- serious dependency failure;
- inability to reach rescue mode.

---

# Part 19 — Rescue and Emergency Service Comparison

The service units were inspected:

```bash
systemctl cat rescue.service
systemctl cat emergency.service
```

Both services use:

```ini
Environment=HOME=/root
WorkingDirectory=-/root
Type=idle
StandardInput=tty-force
```

Both start a systemd sulogin shell.

Rescue service:

```ini
ExecStart=-/usr/lib/systemd/systemd-sulogin-shell rescue
After=sysinit.target plymouth-start.service
```

Emergency service:

```ini
ExecStart=-/usr/lib/systemd/systemd-sulogin-shell emergency
```

## Main Difference

```text
rescue mode
```

expects basic system initialization to have completed.

```text
emergency mode
```

has fewer dependencies and is intended to work earlier in a damaged boot process.

## TTY Requirement

Both services use:

```ini
StandardInput=tty-force
```

This means recovery access is expected through a local console or virtual machine console.

For `phoenix-linux-01`, the correct recovery channel is:

```text
Proxmox console
```

SSH should not be assumed to remain available in rescue or emergency mode.

---

# Part 20 — Why `systemctl isolate` Was Not Used

Both targets contain:

```ini
AllowIsolate=yes
```

This allows a running system to switch to them with commands such as:

```bash
systemctl isolate rescue.target
```

or:

```bash
systemctl isolate emergency.target
```

These commands were intentionally not executed over SSH.

Isolating a recovery target may stop:

- networking;
- SSH;
- nonessential services;
- login sessions;
- mounted remote filesystems.

Running the command remotely could immediately disconnect the administrator.

Recovery-mode testing should be performed through the Proxmox console and only with a clear rollback plan.

---

# Part 21 — Validating `/etc/fstab`

The current filesystem table was checked without rebooting:

```bash
sudo findmnt --verify --verbose
```

For `/mnt/phoenix-data`, the validation confirmed:

```text
target exists
UUID resolves to /dev/sdb1
source exists
filesystem type is ext4
```

Initial summary:

```text
0 parse errors
0 errors
1 warning
```

This showed that the mount configuration contained no syntax or device-resolution error.

---

# Part 22 — Understanding the Swap Warning

The only warning was:

```text
non-bind mount source /swap.img is a directory or regular file
```

This warning appeared because:

```text
/swap.img
```

is a regular file rather than a block device.

Linux supports swap files, so this is not inherently an error.

The active swap state was verified with:

```bash
swapon --show
```

Result:

```text
NAME: /swap.img
TYPE: file
SIZE: 3.1G
USED: 0B
PRIORITY: -2
```

This confirmed that the swap file was valid and active.

## Validation Conclusion

```text
0 parse errors
0 errors
1 expected swap-file warning
```

No boot-blocking `/etc/fstab` problem was present.

---

# Part 23 — Verifying the `/etc/fstab` Backup

The current file and earlier recovery copy were inspected:

```bash
sudo ls -l \
  /etc/fstab \
  /etc/fstab.backup-before-phoenix-data
```

Observed files:

```text
/etc/fstab
/etc/fstab.backup-before-phoenix-data
```

Sizes:

```text
Current file: 735 bytes
Backup file:  657 bytes
```

The backup was created immediately before adding the persistent `/mnt/phoenix-data` entry.

---

# Part 24 — Comparing Current and Backup Configuration

The files were compared with:

```bash
sudo diff -u \
  /etc/fstab.backup-before-phoenix-data \
  /etc/fstab
```

The only added line was:

```text
UUID=44913ffa-70d2-490c-8397-5f17fd2f6db6 /mnt/phoenix-data ext4 defaults 0 2
```

This confirmed that the rollback copy differed only by the new lab-disk mount entry.

## Recovery Value

If the added mount entry caused a boot failure, the known-good backup could be restored from the Proxmox console.

Example recovery command:

```bash
cp /etc/fstab.backup-before-phoenix-data /etc/fstab
```

This command was not executed because the current configuration was valid.

---

# Part 25 — Systemd Reload Warning

A later verification showed:

```text
your fstab has been modified, but systemd still uses the old version
```

This did not mean the file was invalid.

It meant systemd had generated mount-unit state from an earlier version of `/etc/fstab`.

Systemd was reloaded:

```bash
sudo systemctl daemon-reload
```

The file was then revalidated:

```bash
sudo findmnt --verify
```

Final result:

```text
0 parse errors
0 errors
1 warning
```

Only the known swap-file warning remained.

## Operational Lesson

After modifying `/etc/fstab`, use:

```bash
sudo systemctl daemon-reload
```

before relying on systemd-generated mount units or final validation state.

---

# Part 26 — Safe `/etc/fstab` Change Workflow

A safe workflow is:

```text
create backup copy
        ↓
edit one mount entry
        ↓
verify target directory exists
        ↓
verify UUID with blkid or lsblk
        ↓
run findmnt --verify
        ↓
run systemctl daemon-reload
        ↓
test mount without reboot
        ↓
confirm findmnt output
        ↓
reboot only after successful validation
```

Recommended checks:

```bash
sudo findmnt --verify --verbose
```

```bash
lsblk -f
```

```bash
findmnt /mount/point
```

```bash
sudo mount -a
```

`mount -a` should only be used after reviewing the configuration and understanding what it will attempt to mount.

---

# Part 27 — `/etc/fstab` Recovery Workflow

If a bad mount entry prevents normal boot:

1. Open the Proxmox console.
2. Enter emergency or rescue mode.
3. Authenticate for the root recovery shell.
4. Remount root as read-write if required.
5. Inspect `/etc/fstab`.
6. Comment out or correct the failing entry.
7. Restore a known-good backup if appropriate.
8. Validate the file.
9. Reload systemd if operating in a sufficiently complete environment.
10. Reboot.
11. Verify all expected mounts.
12. Review the boot journal.

Example root remount when required:

```bash
mount -o remount,rw /
```

Example validation:

```bash
findmnt --verify
```

Example rollback:

```bash
cp /etc/fstab.backup-before-phoenix-data /etc/fstab
```

These commands must only be used after confirming the correct root filesystem and recovery context.

---

# Part 28 — Complete Boot Flow

The verified startup sequence for this VM is:

```text
Proxmox virtual hardware
        ↓
QEMU firmware environment
        ↓
bootloader
        ↓
vmlinuz-6.8.0-136-generic
        ↓
initrd.img-6.8.0-136-generic
        ↓
virtual storage discovery
        ↓
LVM physical volume /dev/sda3
        ↓
volume group ubuntu-vg
        ↓
logical volume ubuntu-lv
        ↓
ext4 root filesystem mounted read-only
        ↓
systemd PID 1
        ↓
root remounted read-write
        ↓
system services and targets
        ↓
multi-user.target
        ↓
graphical.target
        ↓
SSH and normal administration
```

---

# Boot Troubleshooting Workflow

## Step 1 — Confirm Current State

```bash
systemctl --failed
```

```bash
uptime
```

```bash
findmnt /
```

## Step 2 — Measure Boot Timing

```bash
systemd-analyze
```

```bash
systemd-analyze blame
```

```bash
systemd-analyze critical-chain
```

## Step 3 — Inspect Current Boot Errors

```bash
journalctl -b 0 -p err
```

## Step 4 — Inspect Previous Boot Errors

```bash
journalctl -b -1 -p err
```

## Step 5 — Inspect Kernel Messages

```bash
journalctl -b 0 -k
```

## Step 6 — Verify Boot Artifacts

```bash
uname -r
```

```bash
ls -lh /boot/vmlinuz-* /boot/initrd.img-*
```

## Step 7 — Verify Root Device

```bash
cat /proc/cmdline
```

```bash
findmnt /
```

## Step 8 — Verify Filesystem Configuration

```bash
sudo findmnt --verify --verbose
```

## Step 9 — Verify Storage Layers

```bash
lsblk -f
```

```bash
sudo pvs
sudo vgs
sudo lvs
```

## Step 10 — Use Console Recovery When Needed

Use the Proxmox console if:

- SSH is unavailable;
- networking did not start;
- the system entered emergency mode;
- root filesystem repair is required;
- `/etc/fstab` prevents boot;
- a systemd target transition could stop remote access.

---

# Troubleshooting Scenarios

## Slow Boot

Use:

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
```

Do not disable a service based only on its blame duration.

Confirm that it is on the critical path and understand its purpose first.

## Failed Mount

Check:

```bash
findmnt --verify
lsblk -f
journalctl -b -p err
```

Verify:

- UUID;
- mount point;
- filesystem type;
- mount options;
- device availability.

## Root Filesystem Becomes Read-Only

Check kernel and filesystem messages:

```bash
journalctl -k -p warning
```

A filesystem may remount itself read-only after detecting serious errors.

Do not force normal operation before understanding the cause.

## System Enters Emergency Mode

Common causes include:

- invalid `/etc/fstab`;
- missing required disk;
- failed filesystem check;
- unavailable root or boot device;
- damaged filesystem;
- critical mount dependency failure.

Use the Proxmox console.

## Previous Failure Disappears After Reboot

Use persistent boot history:

```bash
journalctl --list-boots
journalctl -b -1
```

Do not rely only on current-boot logs.

## Historical Error Remains in Journal

A historical failure may remain visible even after correction.

Check current state with:

```bash
systemctl --failed
```

## `findmnt` Warns About `/swap.img`

Confirm active swap:

```bash
swapon --show
```

A swap file is a regular file and may trigger a warning even when working correctly.

## Systemd Uses Old `/etc/fstab` State

Run:

```bash
sudo systemctl daemon-reload
```

Then repeat:

```bash
sudo findmnt --verify
```

---

# Security Considerations

- Do not expose the Proxmox console to untrusted users.
- Recovery shells provide highly privileged access.
- Protect root authentication.
- Do not add insecure kernel parameters without understanding their effect.
- Do not disable security services merely to shorten boot time.
- Preserve historical boot logs for incident investigation.
- Review unexpected changes to `/boot`.
- Verify ownership and permissions of boot-critical files.
- Do not modify GRUB remotely without console access.
- Back up `/etc/fstab` before changes.
- Use UUIDs for persistent mount references.
- Validate configuration before reboot.
- Keep recovery access independent from SSH.

---

# Backup and Recovery Considerations

Important boot-related data to preserve includes:

```text
/etc/fstab
/etc/default/grub
/etc/systemd/system/
/boot
LVM configuration metadata
network configuration
SSH configuration
```

Before risky boot changes:

1. create a Proxmox snapshot;
2. back up the file being changed;
3. confirm console access;
4. record the current kernel;
5. record the current default target;
6. record current mount state;
7. validate syntax;
8. keep rollback commands ready.

A snapshot is useful for short-term rollback but does not replace an independent backup.

---

# GUI-First Recovery Workflow

## Proxmox GUI

Use the Proxmox interface to:

- confirm VM power state;
- open the virtual console;
- inspect virtual disks;
- verify boot order;
- create a snapshot before risky work;
- restart the VM when required;
- observe boot messages without depending on SSH.

## VS Code

Use VS Code to:

- review backed-up configuration files;
- compare configuration versions;
- document recovery procedures;
- inspect Git history;
- prepare corrected configuration safely.

## Terminal

Use the terminal or Proxmox console for:

- journal inspection;
- systemd analysis;
- mount validation;
- LVM inspection;
- root filesystem remounting;
- recovery commands.

## Recommended Model

```text
Proxmox GUI
    → console and VM recovery access

VS Code
    → configuration review and documentation

Linux terminal
    → boot analysis, validation, and repair
```

---

# Final Verification Table

| Check | Result |
|---|---|
| Kernel startup time measured | Passed |
| Userspace startup time measured | Passed |
| Total boot time measured | Passed |
| Slow units listed | Passed |
| Critical boot chain inspected | Passed |
| False bottleneck interpretation avoided | Passed |
| Default target identified | Passed |
| Vendor default target link confirmed | Passed |
| `graphical.target` definition inspected | Passed |
| Persistent boot history listed | Passed |
| Current-boot warnings reviewed | Passed |
| Known lab failure classified | Passed |
| Current serious errors reviewed | Passed |
| Previous boot checked | Passed |
| Active kernel identified | Passed |
| Kernel image verified | Passed |
| Initramfs verified | Passed |
| Kernel command line reviewed | Passed |
| Root device confirmed | Passed |
| Root read-write state confirmed | Passed |
| Early kernel log inspected | Passed |
| KVM virtualization confirmed | Passed |
| NX protection confirmed | Passed |
| LVM activation traced | Passed |
| Current failed-unit state checked | Passed |
| Rescue target inspected | Passed |
| Emergency target inspected | Passed |
| Rescue service inspected | Passed |
| Emergency service inspected | Passed |
| Console recovery requirement documented | Passed |
| `/etc/fstab` validated | Passed |
| Additional mount UUID verified | Passed |
| Swap warning identified | Passed |
| Swap file confirmed active | Passed |
| `/etc/fstab` backup confirmed | Passed |
| Current and backup files compared | Passed |
| Rollback path identified | Passed |
| Systemd reload warning identified | Passed |
| `daemon-reload` completed | Passed |
| Final `/etc/fstab` validation passed | Passed |

---

# Lessons Learned

- Boot time consists of kernel and userspace phases.
- A long `systemd-analyze blame` result does not automatically indicate a boot bottleneck.
- `critical-chain` is more useful for identifying the blocking dependency path.
- Systemd units often start in parallel.
- `graphical.target` can be the default target on a server without a desktop environment.
- Persistent journals allow previous boot analysis.
- Historical journal errors are different from current failed unit state.
- The kernel image and initramfs perform different roles.
- Initramfs is essential when the root filesystem depends on LVM.
- The kernel may initially mount root read-only.
- Systemd later remounts root read-write.
- Early kernel logs confirm virtualization, memory protection, and hardware discovery.
- LVM activation is part of the root-filesystem boot path.
- Rescue mode provides a broader recovery environment.
- Emergency mode provides a more minimal recovery environment.
- Recovery shells require console access and should not be tested casually over SSH.
- `/etc/fstab` should always be validated before reboot.
- A swap-file warning is not necessarily an error.
- `systemctl daemon-reload` is required after relevant configuration changes.
- Configuration backups make recovery faster and safer.
- Proxmox console access is a core recovery requirement for this VM.

---

# Outcome

The `phoenix-linux-01` boot process was analyzed from kernel startup through the final systemd target.

Verified boot performance:

```text
Kernel:    1.834 seconds
Userspace: 3.876 seconds
Total:     5.711 seconds
```

Verified boot components:

```text
Kernel:
  6.8.0-136-generic

Root device:
  /dev/mapper/ubuntu--vg-ubuntu--lv

Root filesystem:
  ext4, read-write during normal operation

Virtualization:
  KVM with QEMU virtual hardware

Default target:
  graphical.target
```

System health:

```text
Current failed units:
  0

Previous boot error-level entries:
  none
```

Filesystem configuration:

```text
/etc/fstab:
  0 parse errors
  0 errors
  1 expected swap-file warning
```

Recovery readiness:

```text
Proxmox console available
Known-good fstab backup available
Additional disk UUID verified
Swap file active
Rescue and emergency modes understood
Rollback workflow documented
```

The system now has a documented boot-analysis and recovery workflow that avoids unnecessary boot configuration changes and preserves a clear recovery path.