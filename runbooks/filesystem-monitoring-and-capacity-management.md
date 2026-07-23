# Filesystem Monitoring and Capacity Management

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Filesystem Monitoring and Capacity Management |
| Subtopic | Disk Usage, Inode Usage, Growth Analysis, and Safe Cleanup Planning |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Status | Completed and verified |

## Purpose

This runbook documents a practical filesystem-capacity investigation on the `phoenix-linux-01` lab virtual machine.

The lab covered:

- establishing disk-capacity baselines;
- measuring inode consumption;
- identifying the largest top-level directories;
- investigating `/var`, `/home`, and application cache growth;
- distinguishing safe cleanup candidates from critical system data;
- identifying stale VS Code Remote Server versions;
- verifying active dependencies before deletion;
- performing targeted cleanup;
- measuring reclaimed capacity;
- demonstrating inode exhaustion with many small files;
- restoring the filesystem to its original baseline.

---

# Environment

```text
System: phoenix-linux-01
Operating system: Ubuntu Server 24.04.1 LTS

Root filesystem:
  /dev/mapper/ubuntu--vg-ubuntu--lv
  Mount point: /

Dedicated lab filesystem:
  /dev/sdb1
  Mount point: /mnt/phoenix-data
```

The lab used:

```text
/home/igor
/var
/mnt/phoenix-data
```

No production service data was removed.

---

# Part 1 — Establishing the Filesystem Baseline

Filesystem capacity was checked with:

```bash
df -h
```

Inode usage was checked with:

```bash
df -i
```

## Root Filesystem Baseline

Observed capacity:

```text
Filesystem: /dev/mapper/ubuntu--vg-ubuntu--lv
Size: 30G
Used: 7.6G
Available: 21G
Usage: 27%
Mount point: /
```

Observed inode state:

```text
Total inodes: 1,966,080
Used inodes: 110,084
Free inodes: 1,855,996
Inode usage: 6%
```

## Lab Filesystem Baseline

Observed capacity:

```text
Filesystem: /dev/sdb1
Size: 3.9G
Used: 44K
Available: 3.7G
Usage: 1%
Mount point: /mnt/phoenix-data
```

Observed inode state:

```text
Total inodes: 262,144
Used inodes: 18
Free inodes: 262,126
Inode usage: 1%
```

## Capacity and Inodes Are Different Resources

Filesystem capacity measures stored data volume.

Inodes represent filesystem objects such as:

- regular files;
- directories;
- symbolic links;
- device nodes.

A filesystem can fail because:

```text
Disk blocks are exhausted
```

or because:

```text
All available inodes are exhausted
```

These conditions are independent.

---

# Part 2 — Understanding `df -h` and `df -i`

## Capacity Check

```bash
df -h /
```

The `-h` option displays values in a human-readable format.

Important columns:

```text
Size
Used
Avail
Use%
Mounted on
```

## Inode Check

```bash
df -i /
```

Important columns:

```text
Inodes
IUsed
IFree
IUse%
Mounted on
```

When a system reports:

```text
No space left on device
```

both commands should be checked:

```bash
df -h
df -i
```

A large amount of free disk space does not guarantee that free inodes remain available.

---

# Part 3 — Analyzing Top-Level Root Usage

The largest first-level directories were identified with:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

## Option Explanation

```text
du
```

Measures directory space usage.

```text
-x
```

Stays on the current filesystem.

This prevented the command from including `/mnt/phoenix-data`.

```text
-h
```

Uses human-readable units.

```text
-d1
```

Limits output to one directory level below `/`.

```text
2>/dev/null
```

Hides non-critical permission errors.

```text
sort -h
```

Sorts human-readable sizes numerically.

## Observed Results

```text
/etc   → 6.2M
/var   → 442M
/home  → 1.9G
/usr   → 2.3G
/      → 7.6G
```

## Interpretation

### `/usr`

Contains installed programs, libraries, and system resources.

It should not be cleaned manually.

Package removal should be performed through the package manager.

### `/var`

Contains variable application and system data.

Common growth areas include:

```text
/var/log
/var/cache
/var/lib
```

### `/home`

Contains user data, application caches, remote-development files, and personal configuration.

### `/etc`

Contains system configuration.

Its size was small and normal.

---

# Part 4 — Investigating `/var`

The first level of `/var` was analyzed with:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Observed usage:

```text
/var/backups → 1.4M
/var/log     → 64M
/var/cache   → 146M
/var/lib     → 231M
/var         → 442M
```

## `/var/log`

Contains active and rotated logs.

Log cleanup should normally be controlled through:

- retention policies;
- `logrotate`;
- journald retention;
- application-specific logging configuration.

Active logs should not be deleted randomly.

## `/var/cache`

Contains regenerable cache data.

Some cache content may be removable, but only after its purpose is identified.

## `/var/lib`

Contains persistent state used by packages and services.

It must not be treated as a general cleanup directory.

## `/var/backups`

Contains system-generated backup copies of selected configuration or metadata.

Its usage was insignificant.

---

# Part 5 — Investigating `/var/cache`

The cache directory was analyzed with:

```bash
sudo du -xhd1 /var/cache 2>/dev/null | sort -h
```

Observed usage:

```text
/var/cache/apparmor  → 1.6M
/var/cache/man       → 2.0M
/var/cache/debconf   → 4.8M
/var/cache/swcatalog → 7.9M
/var/cache/fwupd     → 23M
/var/cache/apt       → 108M
/var/cache           → 146M
```

The largest entry was:

```text
/var/cache/apt
```

## APT Archive Check

Downloaded package archives were checked with:

```bash
sudo du -sh /var/cache/apt/archives
sudo find /var/cache/apt/archives \
  -maxdepth 1 \
  -type f \
  -name '*.deb' \
  | wc -l
```

Result:

```text
Archive size: 20K
Downloaded .deb files: 0
```

This proved that `apt clean` would not reclaim significant capacity.

## APT Cache Contents

The directory was inspected:

```bash
sudo ls -lah /var/cache/apt
```

Observed files:

```text
pkgcache.bin     → 54M
srcpkgcache.bin  → 54M
```

These are binary metadata caches used by APT.

They are regenerable, but removing them was unnecessary because:

```text
Root filesystem free space: approximately 21G
Potential recovery: approximately 108M
```

## Operational Decision

```text
A removable file is not automatically a necessary cleanup target.
```

Cleanup should be driven by:

- actual capacity pressure;
- operational value;
- risk;
- regeneration cost;
- documented retention policy.

---

# Part 6 — Investigating `/var/lib`

The first level of `/var/lib` was inspected:

```bash
sudo du -xhd1 /var/lib 2>/dev/null | sort -h
```

Observed usage:

```text
/var/lib/fwupd             → 2.0M
/var/lib/command-not-found → 3.8M
/var/lib/ubuntu-advantage  → 6.5M
/var/lib/dpkg              → 29M
/var/lib/apt               → 189M
/var/lib                   → 231M
```

## `/var/lib/apt`

Contains APT state and package metadata.

## `/var/lib/dpkg`

Contains the package database and installation state.

This directory is critical.

Manual deletion could corrupt package management and prevent:

- package installation;
- upgrades;
- removals;
- dependency resolution;
- package-status inspection.

## Final Assessment

```text
/var/lib usage was normal.
No cleanup was required.
```

---

# Part 7 — Investigating `/home`

The user home directory was analyzed:

```bash
du -hd1 /home/igor 2>/dev/null | sort -h
```

Observed usage:

```text
/home/igor/.ssh               → 8.0K
/home/igor/.local             → 12K
/home/igor/.cache             → 16K
/home/igor/phoenix-backup-lab → 16K
/home/igor/.vscode-server     → 1.9G
/home/igor                    → 1.9G
```

The dominant consumer was:

```text
/home/igor/.vscode-server
```

This directory is created by VS Code Remote SSH.

It can contain:

- remote VS Code server binaries;
- server versions from previous upgrades;
- remote extensions;
- CLI components;
- logs;
- cached runtime data.

It should not be deleted blindly while a Remote SSH session is active.

---

# Part 8 — Investigating VS Code Remote Server Usage

The first level was checked:

```bash
du -hd1 /home/igor/.vscode-server 2>/dev/null | sort -h
```

Result:

```text
extensions → 8.0K
data       → 1.5M
cli        → 1.8G
total      → 1.9G
```

The CLI subtree was inspected:

```bash
du -hd1 /home/igor/.vscode-server/cli 2>/dev/null | sort -h
```

Result:

```text
/home/igor/.vscode-server/cli/servers → 1.8G
```

Individual server versions were measured:

```bash
du -hd1 /home/igor/.vscode-server/cli/servers \
  2>/dev/null | sort -h
```

Observed versions:

```text
Stable-125df4672b8a6a34975303c6b0baa124e560a4f7 → 598M
Stable-1b6a188127eeaf9194f945eb6eb89a657e93c54c → 598M
Stable-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8 → 598M
```

Total:

```text
1.8G
```

---

# Part 9 — Identifying the Active VS Code Version

Active processes were inspected with:

```bash
ps aux | grep '[v]scode-server'
```

Observed active process:

```text
/home/igor/.vscode-server/code-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8
```

This identified the active server hash:

```text
8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8
```

Corresponding server directory:

```text
Stable-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8
```

This version could not be removed while the active Remote SSH session depended on it.

---

# Part 10 — Reviewing Server Version Dates

The installation directories were inspected:

```bash
ls -ld --time-style=long-iso \
  /home/igor/.vscode-server/cli/servers/Stable-*
```

Observed dates:

```text
Stable-125df... → 2026-07-17 17:29
Stable-8a7abe... → 2026-07-22 16:34
Stable-1b6a18... → 2026-07-22 22:34
```

The newest directory was not automatically removed merely because it was not currently active.

A newer version may be:

- downloaded in preparation for the next connection;
- installed after a client update;
- ready to replace the currently active server;
- valid but not yet launched.

Modification time alone was not sufficient evidence for deletion.

---

# Part 11 — Inspecting Launchers

VS Code launcher files were checked:

```bash
ls -lah /home/igor/.vscode-server/code-*
```

Observed launchers:

```text
code-125df4672b8a6a34975303c6b0baa124e560a4f7 → 32M
code-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8 → 32M
```

There was no launcher for:

```text
1b6a188127eeaf9194f945eb6eb89a657e93c54c
```

This did not prove that the version was invalid.

Its server directory was inspected and contained a complete structure:

```text
server/
package.json
bin/
node_modules/
out/
extensions/
node
product.json
LICENSE
```

It was therefore treated as a valid newer installation.

---

# Part 12 — Identifying Exact VS Code Versions

A broad text search showed many version fields because `product.json` also describes bundled components.

A precise JSON query was then used:

```bash
python3 -c 'import glob,json,os; [print(os.path.basename(os.path.dirname(os.path.dirname(p))), json.load(open(p))["version"]) for p in glob.glob("/home/igor/.vscode-server/cli/servers/Stable-*/server/product.json")]'
```

Confirmed versions:

```text
Stable-125df4672b8a6a34975303c6b0baa124e560a4f7 → 1.129.0
Stable-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8 → 1.129.1
Stable-1b6a188127eeaf9194f945eb6eb89a657e93c54c → 1.130.0
```

## Version Assessment

```text
1.129.0 → stale
1.129.1 → currently active
1.130.0 → newest installed version
```

Only version `1.129.0` was selected for cleanup.

---

# Part 13 — Measuring the Cleanup Candidate

The exact stale objects were measured:

```bash
du -sh \
  /home/igor/.vscode-server/cli/servers/Stable-125df4672b8a6a34975303c6b0baa124e560a4f7 \
  /home/igor/.vscode-server/code-125df4672b8a6a34975303c6b0baa124e560a4f7
```

Observed sizes:

```text
Server directory: 598M
Launcher: 32M
Potential recovery: approximately 630M
```

This was a significant and clearly attributable cleanup target.

---

# Part 14 — Targeted VS Code Cleanup

The stale server version was removed:

```bash
rm -rf \
  /home/igor/.vscode-server/cli/servers/Stable-125df4672b8a6a34975303c6b0baa124e560a4f7
```

Its matching launcher was removed:

```bash
rm \
  /home/igor/.vscode-server/code-125df4672b8a6a34975303c6b0baa124e560a4f7
```

## Safety Conditions

The cleanup was performed only after verifying:

- the target version was not active;
- a newer active version existed;
- an even newer installed version existed;
- the removed directory and launcher used the same hash;
- the expected recovery size was known.

---

# Part 15 — Verifying Remaining VS Code Versions

The remaining servers were measured:

```bash
du -hd1 /home/igor/.vscode-server/cli/servers \
  2>/dev/null | sort -h
```

Result:

```text
Stable-1b6a188127eeaf9194f945eb6eb89a657e93c54c → 598M
Stable-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8 → 598M
Total → 1.2G
```

Remaining launcher:

```text
code-8a7abeba6e03ea3af87bfbce9a1b7e48fed567b8
```

The active version and newest version were preserved.

---

# Part 16 — Measuring Reclaimed Capacity

The VS Code directory and root filesystem were measured again:

```bash
du -sh /home/igor/.vscode-server
df -h /
```

Final results:

```text
VS Code Remote storage:
1.8G → 1.2G
```

```text
Root filesystem used:
7.6G → 7.0G
```

```text
Root filesystem usage:
27% → 25%
```

```text
Available capacity:
21G → 22G
```

Approximately `630M` was reclaimed.

## Cleanup Workflow

```text
measure
    ↓
identify
    ↓
inspect
    ↓
verify active dependencies
    ↓
select one stale target
    ↓
remove
    ↓
measure again
```

---

# Part 17 — Preparing the Inode Lab

A dedicated directory was created:

```bash
mkdir -p /mnt/phoenix-data/inode-lab
```

The initial state was recorded:

```bash
df -h /mnt/phoenix-data
df -i /mnt/phoenix-data
```

Baseline:

```text
Disk used: 48K
Inodes used: 19
```

The test remained isolated on the dedicated lab filesystem.

---

# Part 18 — Creating 1,000 Small Files

A controlled loop created 1,000 empty files:

```bash
for i in $(seq 1 1000); do
  touch "/mnt/phoenix-data/inode-lab/file-$i"
done
```

## Loop Explanation

```text
seq 1 1000
```

Generates numbers from 1 through 1000.

```text
touch
```

Creates an empty file when the file does not already exist.

Each file consumed one inode.

---

# Part 19 — Measuring Inode Growth

After file creation:

```bash
df -h /mnt/phoenix-data
df -i /mnt/phoenix-data
```

Observed result:

```text
Disk usage:
48K → 68K
```

```text
Inodes used:
19 → 1019
```

The disk-space increase was approximately:

```text
20K
```

The inode increase was:

```text
1000
```

This demonstrated that many tiny files can consume large numbers of inodes while using very little data capacity.

---

# Part 20 — Confirming File Count and Directory Size

The number of files was checked:

```bash
find /mnt/phoenix-data/inode-lab \
  -maxdepth 1 \
  -type f \
  | wc -l
```

Result:

```text
1000
```

Directory usage was measured:

```bash
du -sh /mnt/phoenix-data/inode-lab
```

Result:

```text
24K
```

This confirmed:

```text
1,000 files
24K directory usage
1,000 additional inodes
```

---

# Part 21 — Inode Exhaustion Symptoms

When all inodes are used, applications may report:

```text
No space left on device
```

even though `df -h` still shows available capacity.

Common causes include:

- runaway cache directories;
- application session files;
- mail queues;
- temporary files;
- container overlay data;
- build artifacts;
- millions of tiny log files;
- failed cleanup jobs;
- package-manager metadata;
- monitoring data with poor retention.

The diagnostic workflow should include:

```bash
df -h
df -i
```

Potential follow-up analysis:

```bash
sudo find /path -xdev -type f | wc -l
```

or directory-by-directory inode counts.

---

# Part 22 — Cleaning Up the Inode Lab

The dedicated lab directory was removed:

```bash
rm -rf /mnt/phoenix-data/inode-lab
```

The filesystem was checked again:

```bash
df -i /mnt/phoenix-data
df -h /mnt/phoenix-data
```

Final state:

```text
Inodes used:
1019 → 18
```

```text
Disk usage:
68K → 44K
```

The lab filesystem returned to its original baseline.

---

# Final Filesystem State

## Root Filesystem

```text
Size: 30G
Used: 7.0G
Available: 22G
Usage: 25%
```

## Lab Filesystem

```text
Size: 3.9G
Used: 44K
Available: 3.7G
Usage: 1%
```

## Lab Filesystem Inodes

```text
Total: 262,144
Used: 18
Free: 262,126
Usage: 1%
```

---

# Safe Cleanup Principles

## Measure Before Deleting

Use:

```bash
df -h
df -i
du
find
```

Do not begin with deletion.

## Identify Ownership and Purpose

Before deleting a directory, determine:

- which application created it;
- whether a process is using it;
- whether it can be recreated;
- whether it contains state or cache;
- whether a service must be stopped first.

## Avoid Manual Deletion of Critical State

Do not manually remove content from:

```text
/var/lib/dpkg
/var/lib/apt
/usr
/etc
```

without a documented recovery procedure.

## Prefer Application-Aware Cleanup

Examples:

- package managers for package cleanup;
- logrotate for log retention;
- journald settings for journal retention;
- container commands for container data;
- application settings for cache cleanup.

## Remove One Confirmed Target

A targeted removal is safer than broad commands against entire parent directories.

## Measure the Result

After cleanup:

```bash
df -h
df -i
du -sh target
```

Verification is part of the cleanup operation.

---

# Monitoring Thresholds

Example warning thresholds:

```text
Disk capacity warning: 80%
Disk capacity critical: 90%
Inode warning: 80%
Inode critical: 90%
```

Thresholds depend on:

- filesystem size;
- data growth rate;
- service importance;
- recovery time;
- available monitoring;
- operational requirements.

A small filesystem may require intervention earlier because a few gigabytes of growth can fill it quickly.

---

# Growth Analysis

A single measurement shows current usage.

Capacity management also requires trend analysis.

Important questions include:

- How quickly is the filesystem growing?
- Which directory changes most frequently?
- Is growth expected?
- Is retention configured?
- Will free space last until the next maintenance window?
- Is growth caused by data, logs, cache, or temporary files?

A useful repeated measurement might include:

```bash
date
df -h /
du -sh /var/log /var/cache /home/igor
```

For production systems, this should normally be handled by monitoring tools rather than manual checks.

---

# Monitoring and Alerting Considerations

Capacity monitoring can be implemented using:

- Prometheus Node Exporter;
- Grafana dashboards;
- Zabbix;
- Nagios;
- Checkmk;
- Netdata;
- systemd services and timers;
- centralized log and metrics platforms.

Metrics should include:

- filesystem size;
- available bytes;
- used percentage;
- available inodes;
- inode usage percentage;
- growth rate;
- mount status;
- read-only filesystem state.

Alerts should provide enough time for safe investigation before exhaustion occurs.

---

# Troubleshooting

## `df -h` Shows Free Space but Writes Fail

Check:

```bash
df -i
```

The filesystem may have no free inodes.

## `du` and `df` Show Different Usage

Possible reasons:

- deleted files still open by processes;
- reserved filesystem blocks;
- mounted directories;
- sparse files;
- filesystem metadata;
- inaccessible directories;
- snapshots.

Check deleted but open files with:

```bash
sudo lsof +L1
```

This command should be used only when `lsof` is installed.

## A Large File Was Deleted but Space Was Not Reclaimed

A running process may still hold the deleted file open.

Identify the process and restart or reload it safely.

## `/var/log` Is Growing

Investigate:

- which log is growing;
- application error loops;
- missing logrotate rules;
- overly verbose logging;
- journald retention;
- failed compression;
- service restart requirements.

Do not delete active logs without understanding how the application handles its open file descriptor.

## `/home` Is Growing

Common causes:

- remote-development servers;
- language package caches;
- browser or application cache;
- build directories;
- container data;
- downloaded artifacts;
- temporary files;
- old project copies.

## Inode Usage Is High

Look for directories containing very large numbers of files.

Do not assume the largest directory by bytes is also the largest inode consumer.

---

# Security Considerations

- Do not run broad `rm -rf` commands against system paths.
- Confirm every path before pressing Enter.
- Avoid cleanup commands copied from untrusted sources.
- Do not change permissions to bypass cleanup problems.
- Verify active processes before removing runtime data.
- Preserve logs required for incident investigation.
- Protect backup and recovery artifacts.
- Use least privilege.
- Document every meaningful removal.
- Keep rollback options when possible.

---

# Backup and Recovery Considerations

Before deleting important or uncertain data:

1. identify the application owner;
2. determine whether the data is reproducible;
3. create a backup when appropriate;
4. stop dependent services if required;
5. record current ownership and permissions;
6. remove only the selected target;
7. verify service functionality;
8. verify reclaimed space;
9. retain a recovery path.

Cache data may be reproducible.

State data may not be.

The distinction must be understood before cleanup.

---

# GUI-First Workflow

GUI tools can support filesystem analysis, but terminal tools remain important for exact measurement.

## GUI-Appropriate Tasks

VS Code Explorer can help:

- inspect user directories;
- identify visible large project folders;
- review file trees;
- inspect configuration;
- document findings.

Proxmox GUI can help:

- view virtual disk sizes;
- confirm VM storage allocation;
- monitor high-level storage usage;
- create snapshots before risky work.

## Terminal Tasks

The terminal provides reliable measurement for:

```text
df
du
find
stat
ps
```

These tools expose:

- filesystem boundaries;
- inode usage;
- exact size;
- process dependencies;
- hidden directories;
- ownership and permissions.

## Recommended Combined Model

```text
GUI
  → understand the environment and inspect visible structure

Terminal
  → measure capacity, inodes, and process dependencies

GUI documentation
  → record evidence and decisions
```

---

# Final Verification Table

| Check | Result |
|---|---|
| Root capacity baseline collected | Passed |
| Root inode baseline collected | Passed |
| Lab disk capacity baseline collected | Passed |
| Lab disk inode baseline collected | Passed |
| Root top-level usage analyzed | Passed |
| `/var` analyzed | Passed |
| `/var/cache` analyzed | Passed |
| APT archive content checked | Passed |
| APT metadata cache identified | Passed |
| Unnecessary cache deletion avoided | Passed |
| `/var/lib` analyzed | Passed |
| Critical package state identified | Passed |
| `/home` analyzed | Passed |
| VS Code Remote usage identified | Passed |
| Individual server versions measured | Passed |
| Active server hash identified | Passed |
| Version metadata verified | Passed |
| Stale version identified | Passed |
| Cleanup size measured | Passed |
| Targeted cleanup completed | Passed |
| Active version retained | Passed |
| Newest version retained | Passed |
| Reclaimed capacity measured | Passed |
| Inode lab prepared | Passed |
| 1,000 files created | Passed |
| Inode growth measured | Passed |
| Disk-growth difference measured | Passed |
| File count verified | Passed |
| Inode lab removed | Passed |
| Inode baseline restored | Passed |
| Lab disk capacity restored | Passed |

---

# Lessons Learned

- Disk capacity and inode capacity are separate resources.
- Both `df -h` and `df -i` are required for complete capacity checks.
- `du -x` prevents analysis from crossing into other mounted filesystems.
- `/usr` and `/var/lib` should not be cleaned manually.
- Cache data should be identified before removal.
- Potential cleanup value should be compared with available capacity.
- An available cleanup candidate does not mean cleanup is necessary.
- Application version directories must be correlated with active processes.
- File dates alone are not enough to determine whether a version is stale.
- Structured metadata is more reliable than broad text matching.
- Cleanup should target one verified object at a time.
- Capacity should be measured again after cleanup.
- Thousands of tiny files can exhaust inodes without consuming much disk space.
- `No space left on device` may indicate inode exhaustion.
- Temporary lab data must be removed after testing.
- Safe capacity management is based on evidence, not random deletion.

---

# Outcome

A complete filesystem-capacity investigation and cleanup workflow was successfully completed on `phoenix-linux-01`.

The lab produced the following results:

```text
Root filesystem usage:
27% → 25%

Root available capacity:
21G → 22G

VS Code Remote storage:
1.8G → 1.2G

Reclaimed capacity:
approximately 630M
```

The inode lab demonstrated:

```text
1,000 empty files
approximately 20K additional disk usage
1,000 additional inodes
```

After cleanup, the lab filesystem returned to:

```text
Disk usage: 44K
Inodes used: 18
```

The system now has a documented process for capacity baselining, growth analysis, dependency verification, targeted cleanup, inode troubleshooting, and post-cleanup validation.