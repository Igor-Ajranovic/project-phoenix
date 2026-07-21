# Storage, LVM, and Persistent Mounts

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Storage and Filesystems |
| Subtopic | LVM Expansion, QEMU Guest Agent, and Persistent UUID-Based Mounts |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Hypervisor | Proxmox VE |
| Status | Completed and verified |

## Purpose

This runbook documents a complete storage administration lab on `phoenix-linux-01`.

The lab included:

- inspecting the existing disk and LVM layout;
- identifying unused LVM capacity;
- extending the root logical volume and ext4 filesystem online;
- installing and verifying the QEMU Guest Agent;
- attaching a dedicated 4 GiB lab disk;
- creating a GPT partition;
- creating an ext4 filesystem;
- mounting the filesystem manually;
- assigning user ownership;
- configuring a persistent UUID-based mount in `/etc/fstab`;
- validating the configuration before reboot;
- verifying persistence after reboot.

## Safety Controls

The following safety controls were used:

- The work was performed on a dedicated lab VM.
- The production Lenovo2 system was not modified.
- A Proxmox snapshot was created before the LVM expansion.
- The existing root filesystem was inspected before any changes.
- The new data disk was separate from the operating system disk.
- The `/etc/fstab` file was backed up before editing.
- The new `/etc/fstab` entry was validated before reboot.
- Proxmox console access remained available as a recovery method.

---

# Part 1 — Existing Storage Inspection

## Disk and Filesystem Layout

The storage layout was inspected with:

```bash
lsblk -f
```

The size and device hierarchy were reviewed with:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Observed layout:

```text
sda                         32G disk
├─sda1                       1M part
├─sda2                       2G part ext4        /boot
└─sda3                      30G part LVM2_member
  └─ubuntu--vg-ubuntu--lv   15G lvm  ext4        /
```

The structure was:

```text
/dev/sda
  ↓
/dev/sda3
  ↓
LVM physical volume
  ↓
ubuntu-vg
  ↓
ubuntu-lv
  ↓
ext4
  ↓
/
```

## LVM Volume Group Inspection

The volume group was inspected with:

```bash
sudo vgs
```

Observed result:

```text
VG        VSize    VFree
ubuntu-vg <30.00g  15.00g
```

This confirmed that approximately 15 GiB was still unallocated inside the volume group.

## Logical Volume Inspection

The logical volume was inspected with:

```bash
sudo lvs
```

Observed result:

```text
LV        VG        LSize
ubuntu-lv ubuntu-vg <15.00g
```

## Root Filesystem Verification

The root filesystem was inspected with:

```bash
findmnt /
```

Observed result:

```text
TARGET SOURCE                            FSTYPE OPTIONS
/      /dev/mapper/ubuntu--vg-ubuntu--lv ext4   rw,relatime
```

This confirmed that the root filesystem was:

- stored on the LVM logical volume;
- formatted as ext4;
- mounted read-write;
- eligible for online expansion.

---

# Part 2 — Root LVM Expansion

## Recovery Snapshot

A Proxmox snapshot was created before the storage change.

Snapshot name:

```text
before-lvm-expansion
```

Description:

```text
Recovery point before extending the root logical volume and ext4 filesystem.
```

## Logical Volume and Filesystem Expansion

The logical volume and ext4 filesystem were expanded in one command:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

Command explanation:

```text
-l +100%FREE
```

Uses all remaining free extents in the volume group.

```text
-r
```

Automatically resizes the filesystem after extending the logical volume.

```text
/dev/ubuntu-vg/ubuntu-lv
```

Specifies the root logical volume.

The logical volume grew from approximately 15 GiB to 30 GiB.

The ext4 filesystem was resized online without rebooting the server.

## Expansion Verification

The result was verified with:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -h /
sudo vgs
```

Final state:

```text
Logical volume size: 30G
Root filesystem size: 30G
Available root space: approximately 22G
Root filesystem usage: approximately 25%
Volume group free space: 0
```

## Key Concept

LVM and filesystem capacity are separate layers.

```text
Virtual disk
  ↓
Partition
  ↓
LVM physical volume
  ↓
Volume group
  ↓
Logical volume
  ↓
Filesystem
  ↓
Mount point
```

Expanding only the logical volume does not automatically expand the filesystem unless the filesystem is resized as well.

The `-r` option handled both operations.

---

# Part 3 — QEMU Guest Agent

## Initial Problem

During the Proxmox snapshot, the following warning appeared:

```text
skipping guest filesystem freeze - agent configured but not running
```

The QEMU Guest Agent option was enabled in Proxmox, but the guest package was not installed inside Ubuntu.

## Agent Status Check

The initial check was:

```bash
systemctl status qemu-guest-agent --no-pager
```

Initial result:

```text
Unit qemu-guest-agent.service could not be found.
```

## Agent Installation

The package was installed with:

```bash
sudo apt install qemu-guest-agent
```

The installation also added:

```text
liburing2
```

## Agent Verification

The service was checked again:

```bash
systemctl status qemu-guest-agent --no-pager
```

Confirmed result:

```text
Loaded: loaded
Active: active (running)
Main process: qemu-ga
```

## Proxmox Verification

The Proxmox Summary page displayed:

```text
192.168.20.101
fe80::be24:11ff:febd:df49
```

This confirmed successful communication between Proxmox and the VM.

## Practical Benefits

The QEMU Guest Agent provides:

- guest IP reporting;
- improved shutdown and reboot operations;
- filesystem freeze support during snapshots and backups;
- more consistent snapshot operations;
- improved Proxmox guest information.

---

# Part 4 — Dedicated Lab Disk

## Disk Creation in Proxmox

A dedicated virtual disk was added to VM 120.

Configuration:

```text
Bus/Device: SCSI 1
Storage: ssd512-thin
Disk size: 4 GiB
Discard: enabled
IO thread: enabled
Backup: enabled
```

The new disk appeared inside Ubuntu as:

```text
/dev/sdb
```

Verification:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Observed result:

```text
sdb 4G disk
```

The disk initially had:

- no partition table;
- no partition;
- no filesystem;
- no UUID;
- no mount point.

---

# Part 5 — GPT Partition Creation

A GPT partition table and one full-size partition were created:

```bash
sudo parted /dev/sdb --script mklabel gpt mkpart primary ext4 0% 100%
```

The result was verified with:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sdb
```

Observed result:

```text
sdb      4G disk
└─sdb1   4G part
```

The partition existed but still had no filesystem.

---

# Part 6 — ext4 Filesystem Creation

An ext4 filesystem was created with the label `phoenix-data`:

```bash
sudo mkfs.ext4 -L phoenix-data /dev/sdb1
```

The filesystem identity was verified with:

```bash
lsblk -f /dev/sdb
sudo blkid /dev/sdb1
```

Confirmed result:

```text
Device: /dev/sdb1
Filesystem: ext4
Label: phoenix-data
UUID: 44913ffa-70d2-490c-8397-5f17fd2f6db6
```

## UUID and PARTUUID

The filesystem had both a UUID and a PARTUUID.

```text
UUID
```

Identifies the filesystem.

```text
PARTUUID
```

Identifies the partition entry in the partition table.

For this lab, the filesystem UUID was used in `/etc/fstab`.

---

# Part 7 — Temporary Mount

## Mount Point Creation

The mount point was created:

```bash
sudo mkdir -p /mnt/phoenix-data
```

## Manual Mount

The filesystem was mounted manually:

```bash
sudo mount /dev/sdb1 /mnt/phoenix-data
```

## Mount Verification

The active mount was verified with:

```bash
findmnt /mnt/phoenix-data
df -h /mnt/phoenix-data
```

Observed result:

```text
/mnt/phoenix-data /dev/sdb1 ext4 rw,relatime
```

Capacity:

```text
Size: approximately 3.9G
Available: approximately 3.7G
Usage: approximately 1%
```

The difference between the virtual disk size and available filesystem capacity is normal because partitioning and filesystem metadata consume some space.

---

# Part 8 — Ownership and Write Access

## Initial Ownership

The root directory of the new filesystem was initially owned by:

```text
root:root
```

The regular user `igor` could not create files there.

## Ownership Change

Ownership was assigned to `igor`:

```bash
sudo chown igor:igor /mnt/phoenix-data
```

The result was verified with:

```bash
ls -ld /mnt/phoenix-data
```

## Write Test

Write access was tested with:

```bash
touch /mnt/phoenix-data/test-file.txt
ls -l /mnt/phoenix-data
```

Confirmed result:

```text
-rw-rw-r-- 1 igor igor 0 test-file.txt
```

The `lost+found` directory remained owned by `root`.

This is normal for ext4 filesystems.

The directory is used during filesystem recovery operations.

---

# Part 9 — `/etc/fstab` Persistent Mount

## Existing `/etc/fstab` Review

The file was inspected with:

```bash
cat /etc/fstab
```

Existing entries included:

- root filesystem;
- `/boot`;
- swap file.

The active mount sources were compared with:

```bash
sudo blkid
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /boot
```

## `/etc/fstab` Backup

A backup was created before editing:

```bash
sudo cp /etc/fstab /etc/fstab.backup-before-phoenix-data
```

## Persistent Mount Entry

The following line was added to `/etc/fstab`:

```fstab
UUID=44913ffa-70d2-490c-8397-5f17fd2f6db6 /mnt/phoenix-data ext4 defaults 0 2
```

Column explanation:

| Field | Meaning |
|---|---|
| `UUID=...` | Stable filesystem identifier |
| `/mnt/phoenix-data` | Mount point |
| `ext4` | Filesystem type |
| `defaults` | Standard mount options |
| `0` | Disable dump backup flag |
| `2` | Filesystem check order after root |

## Configuration Validation

The configuration was validated with:

```bash
sudo findmnt --verify --verbose
```

The new entry was successfully translated to:

```text
/dev/sdb1
```

The only persistent warning concerned `/swap.img`, which is a regular file used as swap.

There were:

```text
0 parse errors
0 errors
```

## systemd Reload

After editing `/etc/fstab`, systemd was reloaded:

```bash
sudo systemctl daemon-reload
```

The configuration was verified again:

```bash
sudo findmnt --verify
```

Final result:

```text
0 parse errors
0 errors
1 warning
```

The remaining warning was only related to the normal swap file configuration.

---

# Part 10 — Persistent Mount Testing

## Test Without Reboot

The filesystem was unmounted:

```bash
sudo umount /mnt/phoenix-data
```

All `/etc/fstab` entries were then mounted:

```bash
sudo mount -a
```

The mount was verified:

```bash
findmnt /mnt/phoenix-data
```

Confirmed result:

```text
/mnt/phoenix-data /dev/sdb1 ext4 rw,relatime
```

This confirmed that the `/etc/fstab` entry worked without requiring a reboot.

## Reboot Persistence Test

The VM was rebooted:

```bash
sudo reboot
```

After reconnecting through SSH, the mount was verified with:

```bash
findmnt /mnt/phoenix-data
ls -l /mnt/phoenix-data
```

Confirmed results:

- `/dev/sdb1` mounted automatically;
- the mount point was `/mnt/phoenix-data`;
- the filesystem was ext4;
- the test file remained present;
- ownership remained `igor:igor`;
- the mount survived the reboot.

---

# Final Storage State

## System Disk

```text
/dev/sda
├─/dev/sda2
│  └─ext4
│     └─/boot
└─/dev/sda3
   └─LVM physical volume
      └─ubuntu-vg
         └─ubuntu-lv
            └─ext4
               └─/
```

## Lab Data Disk

```text
/dev/sdb
└─/dev/sdb1
   └─ext4
      ├─Label: phoenix-data
      ├─UUID: 44913ffa-70d2-490c-8397-5f17fd2f6db6
      └─Mount point: /mnt/phoenix-data
```

## Final Verification Table

| Check | Result |
|---|---|
| Existing storage inspected | Passed |
| Free LVM space identified | Passed |
| Proxmox snapshot created | Passed |
| Logical volume expanded | Passed |
| ext4 filesystem expanded online | Passed |
| Root capacity increased | Passed |
| QEMU Guest Agent installed | Passed |
| QEMU Guest Agent active | Passed |
| IP visible in Proxmox | Passed |
| Dedicated 4 GiB disk attached | Passed |
| GPT partition created | Passed |
| ext4 filesystem created | Passed |
| Filesystem label assigned | Passed |
| UUID verified | Passed |
| Temporary mount tested | Passed |
| User ownership configured | Passed |
| User write access verified | Passed |
| `/etc/fstab` backed up | Passed |
| UUID-based persistent mount added | Passed |
| `/etc/fstab` validated | Passed |
| `mount -a` test passed | Passed |
| Reboot persistence test passed | Passed |

---

# Recovery Procedures

## Recovering from a Bad `/etc/fstab` Entry

If the system fails to boot because of an invalid `/etc/fstab` entry:

1. Open the VM console in Proxmox.
2. Enter recovery or emergency mode if necessary.
3. Remount the root filesystem read-write if required.
4. Restore the backup:

```bash
sudo cp /etc/fstab.backup-before-phoenix-data /etc/fstab
```

5. Validate the file:

```bash
sudo findmnt --verify
```

6. Reboot:

```bash
sudo reboot
```

## Removing the Lab Mount Safely

First stop processes using the mount point.

Then unmount it:

```bash
sudo umount /mnt/phoenix-data
```

Remove or comment out the related `/etc/fstab` line.

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Verify:

```bash
sudo findmnt --verify
```

The virtual disk should only be removed from Proxmox after it is unmounted and removed from `/etc/fstab`.

## Recovering from an LVM Expansion Problem

If the VM becomes unbootable after the LVM operation:

1. Open the Proxmox console.
2. Review boot errors.
3. Confirm logical volume availability.
4. Verify filesystem state.
5. Restore the `before-lvm-expansion` snapshot if the system cannot be safely repaired.

---

# Key Concepts

## Disk

A complete physical or virtual storage device.

Example:

```text
/dev/sdb
```

## Partition

A defined section of a disk.

Example:

```text
/dev/sdb1
```

## Filesystem

The structure used to store files and directories.

Example:

```text
ext4
```

## Mount Point

A directory through which a filesystem becomes accessible.

Example:

```text
/mnt/phoenix-data
```

## UUID

A stable filesystem identifier.

It is more reliable than device names such as `/dev/sdb1`, because device names may change.

## `/etc/fstab`

Defines filesystems that should be mounted automatically.

## LVM

Adds a flexible storage layer between partitions and filesystems.

LVM allows logical volumes to be resized more easily than traditional fixed partitions.

## QEMU Guest Agent

Provides communication between the Proxmox host and the Ubuntu guest VM.

---

# Lessons Learned

- Disk size, partition size, logical volume size, and filesystem size are separate concepts.
- Free space inside an LVM volume group is not the same as free space inside a filesystem.
- A logical volume and its filesystem must both be expanded.
- ext4 supports online expansion.
- Snapshots should be created before risky storage changes.
- QEMU Guest Agent must be installed inside the guest VM, even when enabled in Proxmox.
- New filesystems are normally owned by `root`.
- Ownership controls which users and services can write to mounted storage.
- Manual mounts do not survive reboot.
- `/etc/fstab` provides persistent mounts.
- UUID-based mounts are safer than device-name-based mounts.
- `/etc/fstab` must be backed up and validated before reboot.
- `mount -a` provides a safe test before reboot.
- Successful reboot verification is part of the storage change.

## Outcome

The `phoenix-linux-01` VM now has:

- a fully expanded 30 GiB root filesystem;
- working Proxmox guest-agent integration;
- a dedicated 4 GiB data disk;
- an ext4 filesystem labeled `phoenix-data`;
- a persistent UUID-based mount;
- verified user write access;
- documented recovery procedures.