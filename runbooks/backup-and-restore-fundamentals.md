# Backup and Restore Fundamentals

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Backup and Restore Fundamentals |
| Subtopic | Archive Creation, Integrity Verification, Isolated Restore, and Recovery |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Status | Completed and verified |

## Purpose

This runbook documents a complete backup and restore workflow using a dedicated Linux lab environment.

The objectives were to:

- create realistic source data;
- create a compressed archive;
- inspect archive contents without extraction;
- verify gzip integrity;
- generate and verify a SHA-256 checksum;
- simulate data loss safely;
- restore into an isolated location;
- verify content, ownership, permissions, and timestamps;
- compare restored data with archive content;
- restore to the original location;
- remove temporary restore data;
- preserve a verified backup artifact.

---

# Environment

```text
System: phoenix-linux-01
Source: /home/igor/phoenix-backup-lab
Backup destination: /mnt/phoenix-data/backups
Restore test location: /home/igor/phoenix-restore-test
Archive: /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
Checksum: /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz.sha256
```

The lab used only dedicated test data.

No production service data or system configuration was modified.

---

# Part 1 — Preparing Isolated Lab Locations

The following locations were checked:

```bash
ls -ld /home/igor/phoenix-backup-lab \
       /mnt/phoenix-data/backups \
       /home/igor/phoenix-restore-test
```

Initial result:

```text
No such file or directory
```

This confirmed that the paths were available for the lab.

The directories were then created:

```bash
mkdir -p /home/igor/phoenix-backup-lab \
         /mnt/phoenix-data/backups \
         /home/igor/phoenix-restore-test
```

## Line Continuation

The backslash:

```text
\
```

at the end of a command line tells the shell that the command continues on the next line.

Example:

```bash
ls -ld /path/one \
       /path/two \
       /path/three
```

This is equivalent to:

```bash
ls -ld /path/one /path/two /path/three
```

The backslash must be the final character on the line.

---

# Part 2 — Creating Source Data

A realistic test dataset was created:

```bash
mkdir -p /home/igor/phoenix-backup-lab/config
```

Configuration content:

```bash
printf 'service_name=phoenix-demo\nbackup_enabled=true\n' \
  > /home/igor/phoenix-backup-lab/config/app.conf
```

Documentation content:

```bash
printf '# Phoenix Backup Lab\n\nThis file is used to verify backup and restore integrity.\n' \
  > /home/igor/phoenix-backup-lab/README.md
```

## Why `printf` Was Used

`printf` provides predictable formatting.

The sequence:

```text
\n
```

always represents a newline.

This makes `printf` suitable for:

- scripts;
- configuration generation;
- reproducible test data;
- precise multiline output.

`echo` remains useful for simple interactive output, but its handling of escape sequences can vary between shells and implementations.

## Source Verification

The files were checked with:

```bash
find /home/igor/phoenix-backup-lab \
  -maxdepth 2 \
  -type f \
  -exec ls -l {} \;
```

Observed state:

```text
-rw-rw-r-- igor igor 46 app.conf
-rw-rw-r-- igor igor 80 README.md
```

---

# Part 3 — Creating the Compressed Archive

The backup archive was created with:

```bash
tar -czvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  -C /home/igor \
  phoenix-backup-lab
```

## Option Explanation

```text
-c
```

Creates a new archive.

```text
-z
```

Compresses the archive with gzip.

```text
-v
```

Displays processed files.

```text
-f
```

Defines the archive filename.

```text
-C /home/igor
```

Changes the working directory before adding content.

This prevented absolute paths from being stored in the archive.

The archive contained:

```text
phoenix-backup-lab/
phoenix-backup-lab/config/
phoenix-backup-lab/config/app.conf
phoenix-backup-lab/README.md
```

---

# Part 4 — Inspecting Archive Contents

The archive was inspected without extraction:

```bash
tar -tzvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

## Option Explanation

```text
-t
```

Lists archive contents.

```text
-z
```

Reads gzip-compressed data.

```text
-v
```

Displays metadata.

```text
-f
```

Specifies the archive file.

Observed metadata:

```text
drwxrwxr-x igor/igor phoenix-backup-lab/
drwxrwxr-x igor/igor phoenix-backup-lab/config/
-rw-rw-r-- igor/igor 46 phoenix-backup-lab/config/app.conf
-rw-rw-r-- igor/igor 80 phoenix-backup-lab/README.md
```

This confirmed that the archive preserved:

- directory structure;
- ownership metadata;
- file modes;
- file sizes;
- timestamps.

## Operational Lesson

Command output must not be copied back into the shell as commands.

During the lab, archive listing output was accidentally interpreted as shell input, producing messages such as:

```text
Is a directory
Permission denied
```

No data was damaged.

The archive command was simply repeated correctly.

---

# Part 5 — Testing Gzip Integrity

The compressed archive was tested with:

```bash
gzip -t /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

Successful execution produced no output.

This confirmed that the gzip stream was structurally valid.

## Important Distinction

`gzip -t` verifies the compressed file structure.

It does not prove that:

- all required source files were included;
- the correct files were backed up;
- a restore will succeed;
- restored application data will be usable.

Those requirements must be validated separately.

---

# Part 6 — Creating a SHA-256 Checksum

A checksum file was created:

```bash
sha256sum /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  > /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz.sha256
```

Recorded checksum:

```text
bd369e41fbe1112a9d79e4e0d4cf8c9dec4ae6a14c05c477299ddcbc885ca282
```

## Checksum Concept

A checksum is a digital fingerprint of a file.

If even one byte changes, the SHA-256 value will almost certainly change.

Checksums are useful for detecting:

- corruption;
- unintended modification;
- incomplete file transfer;
- storage errors;
- mismatch between source and destination copies.

## Checksum Verification

The checksum was verified with:

```bash
cd /mnt/phoenix-data/backups
sha256sum -c phoenix-backup-lab.tar.gz.sha256
```

Result:

```text
/mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz: OK
```

This confirmed that the archive matched the recorded checksum.

## Security Limitation

A checksum verifies integrity, not authenticity.

If an attacker can modify both the archive and the checksum file, a normal checksum comparison may still pass.

For stronger authenticity guarantees, use:

- signed checksums;
- trusted metadata;
- protected backup storage;
- cryptographic signatures.

---

# Part 7 — Simulating Data Loss

The source directory was intentionally removed:

```bash
rm -rf /home/igor/phoenix-backup-lab
```

The deletion was verified:

```bash
ls -ld /home/igor/phoenix-backup-lab
```

Result:

```text
No such file or directory
```

The backup archive remained intact in:

```text
/mnt/phoenix-data/backups
```

This created a controlled recovery scenario.

---

# Part 8 — Restoring to an Isolated Location

The archive was restored into a dedicated test directory:

```bash
tar -xzvf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  -C /home/igor/phoenix-restore-test
```

## Option Explanation

```text
-x
```

Extracts archive contents.

```text
-z
```

Reads gzip-compressed data.

```text
-v
```

Displays restored paths.

```text
-f
```

Specifies the archive.

```text
-C
```

Defines the extraction destination.

Restored structure:

```text
phoenix-backup-lab/
phoenix-backup-lab/config/
phoenix-backup-lab/config/app.conf
phoenix-backup-lab/README.md
```

---

# Part 9 — Why Isolated Restore Testing Matters

A safe restore workflow usually begins in an isolated location.

This reduces the risk of:

- overwriting current data;
- restoring incorrect versions;
- applying unexpected permissions;
- introducing untrusted archive contents;
- disrupting a running service;
- losing the ability to roll back.

Recommended workflow:

```text
verify checksum
        ↓
inspect archive
        ↓
restore to isolated location
        ↓
verify content and metadata
        ↓
stop dependent service if required
        ↓
backup current live data
        ↓
restore to production location
        ↓
test service
        ↓
retain rollback option
```

Direct production restore may be appropriate during disaster recovery only when the procedure has already been tested and documented.

---

# Part 10 — Verifying Restored Metadata

The restored content was inspected with:

```bash
find /home/igor/phoenix-restore-test/phoenix-backup-lab \
  -maxdepth 2 \
  -exec ls -ld {} \;
```

Observed state:

```text
drwxrwxr-x igor igor phoenix-backup-lab
drwxrwxr-x igor igor config
-rw-rw-r-- igor igor app.conf
-rw-rw-r-- igor igor README.md
```

The restore preserved:

- owner `igor`;
- group `igor`;
- directory mode `775`;
- file mode `664`;
- original directory hierarchy;
- original timestamps.

---

# Part 11 — Verifying Restored Content

The files were read:

```bash
cat /home/igor/phoenix-restore-test/phoenix-backup-lab/config/app.conf
cat /home/igor/phoenix-restore-test/phoenix-backup-lab/README.md
```

Restored configuration:

```text
service_name=phoenix-demo
backup_enabled=true
```

Restored documentation:

```text
# Phoenix Backup Lab

This file is used to verify backup and restore integrity.
```

This confirmed that the expected logical content was present.

---

# Part 12 — Reading Content Directly from the Archive

The configuration file was displayed without extraction:

```bash
tar -xOzf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz \
  phoenix-backup-lab/config/app.conf
```

## `-O` Option

The uppercase `-O` option sends extracted file content to standard output instead of writing it to disk.

Result:

```text
service_name=phoenix-demo
backup_enabled=true
```

This matched the restored file.

---

# Part 13 — Byte-for-Byte Comparison

The archive content and restored file were compared with:

```bash
diff \
  <(tar -xOzf /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz phoenix-backup-lab/config/app.conf) \
  /home/igor/phoenix-restore-test/phoenix-backup-lab/config/app.conf
```

The command produced no output.

For `diff`, no output means the compared content is identical.

This confirmed byte-for-byte content integrity for `app.conf`.

## Process Substitution

The syntax:

```text
<(command)
```

allows command output to be treated as a temporary file-like input.

This enabled `diff` to compare archive content directly with the restored file.

---

# Part 14 — Restoring to the Original Location

After successful isolated verification, the data was copied back:

```bash
cp -a /home/igor/phoenix-restore-test/phoenix-backup-lab \
      /home/igor/
```

## Archive Mode

The `-a` option preserves as much metadata as possible, including:

- directory structure;
- file permissions;
- timestamps;
- symbolic links;
- ownership when permitted.

The original location was then checked:

```bash
ls -ld /home/igor/phoenix-backup-lab
find /home/igor/phoenix-backup-lab \
  -maxdepth 2 \
  -type f \
  -exec ls -l {} \;
```

Confirmed state:

```text
/home/igor/phoenix-backup-lab
```

Owner and group:

```text
igor:igor
```

Permissions:

```text
directories: 775
files: 664
```

Files:

```text
config/app.conf
README.md
```

---

# Part 15 — Cleaning Up the Test Restore

The isolated test copy was removed:

```bash
rm -rf /home/igor/phoenix-restore-test/phoenix-backup-lab
```

Final verification:

```bash
ls -ld /home/igor/phoenix-restore-test/phoenix-backup-lab \
       /home/igor/phoenix-backup-lab \
       /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

Confirmed result:

```text
Test restore copy: removed
Original source: restored
Backup archive: retained
```

The final archive remained:

```text
/mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz
```

Its final size was approximately:

```text
308 bytes
```

---

# Complete Recovery Workflow

```text
Create source data
        ↓
Create tar.gz archive
        ↓
Inspect archive contents
        ↓
Test gzip integrity
        ↓
Generate SHA-256 checksum
        ↓
Verify checksum
        ↓
Simulate source loss
        ↓
Restore to isolated location
        ↓
Verify ownership and permissions
        ↓
Verify file content
        ↓
Compare archive and restored content
        ↓
Restore to original location
        ↓
Remove temporary restore data
        ↓
Retain verified backup
```

---

# Backup Validation Layers

A reliable backup process should use multiple validation layers.

## Layer 1 — Archive Exists

```bash
ls -l backup.tar.gz
```

This proves only that a file exists.

## Layer 2 — Archive Structure Is Readable

```bash
tar -tzf backup.tar.gz
```

This confirms that tar can read the archive table of contents.

## Layer 3 — Compression Integrity

```bash
gzip -t backup.tar.gz
```

This verifies the gzip stream.

## Layer 4 — Checksum Integrity

```bash
sha256sum -c backup.tar.gz.sha256
```

This verifies that the archive has not changed since the checksum was generated.

## Layer 5 — Content Inspection

```bash
tar -xOzf backup.tar.gz path/to/file
```

This verifies specific logical content.

## Layer 6 — Isolated Restore

Extract to a test location and inspect the result.

## Layer 7 — Functional Restore Test

For application data, start or validate the dependent application against the restored data.

A backup is not fully trusted until restore behavior has been tested.

---

# `tar` Command Reference

## Create a gzip-compressed archive

```bash
tar -czf backup.tar.gz source-directory
```

## List archive contents

```bash
tar -tzf backup.tar.gz
```

## Extract an archive

```bash
tar -xzf backup.tar.gz
```

## Extract to a specific directory

```bash
tar -xzf backup.tar.gz -C /restore/location
```

## Read one file without extracting it

```bash
tar -xOzf backup.tar.gz path/inside/archive
```

## Create an archive using a clean relative path

```bash
tar -czf backup.tar.gz -C /source-parent source-directory
```

---

# GUI-First Workflow

VS Code can support parts of this workflow.

## Suitable GUI Tasks

VS Code Explorer can be used to:

- create source directories;
- create and edit test files;
- inspect restored directory structure;
- review file content;
- compare text files visually;
- document results.

## Tasks Better Performed in the Terminal

The terminal is more reliable for:

- creating `tar.gz` archives;
- inspecting archive metadata;
- verifying gzip integrity;
- generating checksums;
- validating checksums;
- preserving permissions and timestamps;
- extracting to controlled destinations;
- performing byte-for-byte comparisons.

## Recommended Combined Workflow

```text
VS Code Explorer
    → create and inspect source data

VS Code integrated terminal
    → archive, verify, checksum, restore

VS Code Diff
    → visually compare text files
```

This preserves a GUI-first learning approach without hiding the actual Linux backup mechanisms.

---

# Security Considerations

- Do not blindly extract untrusted archives as `root`.
- Inspect archive paths before extraction.
- Avoid archives containing unexpected absolute paths.
- Watch for symbolic links pointing outside the restore location.
- Verify file ownership and permissions after restore.
- Protect checksum files from unauthorized modification.
- Store backups on media separate from the source system.
- Encrypt backups that contain sensitive data.
- Do not store secrets in plain-text lab archives.
- Restrict backup directory permissions.
- Test restores using least privilege.
- Stop dependent services before replacing live application data.
- Preserve rollback copies before overwriting production data.

---

# Backup Strategy Considerations

This lab created one local archive on a second filesystem.

That is useful for learning and local recovery, but it is not a complete backup strategy.

A production-quality backup design should consider the 3-2-1 principle:

```text
3 copies of data
2 different storage media
1 copy off-site
```

A stronger design may include:

- local snapshot;
- local backup disk;
- remote or off-site copy;
- encryption;
- retention policy;
- automated verification;
- monitoring and alerts;
- regular restore tests.

---

# Backup vs Snapshot

A snapshot and a backup are not the same.

## Snapshot

A snapshot usually:

- is fast;
- captures a point-in-time filesystem or VM state;
- may depend on the same storage system;
- is useful for short-term rollback.

## Backup

A backup should:

- exist independently from the source;
- survive source deletion or failure;
- have defined retention;
- support restore to another location;
- be regularly verified.

Snapshots can be part of a backup strategy, but they should not be the only protection.

---

# Recovery Considerations

Before restoring production data:

1. identify the correct backup;
2. verify its checksum;
3. inspect the archive;
4. stop the dependent service;
5. preserve current live data;
6. restore into an isolated location;
7. verify ownership and permissions;
8. compare important files;
9. move or copy into production;
10. start the service;
11. perform functional validation;
12. retain a rollback copy.

---

# Troubleshooting

## Archive Creation Fails

Check:

- source path;
- destination directory;
- available disk space;
- write permissions;
- spelling of `tar` options.

## Archive Is Empty

List contents:

```bash
tar -tzf backup.tar.gz
```

Verify that the correct source directory was specified.

## Checksum Verification Fails

Possible causes:

- archive corruption;
- incomplete copy;
- archive modification;
- incorrect checksum file;
- path mismatch inside the checksum file.

Do not proceed with restore until the mismatch is understood.

## Restore Fails with Permission Denied

Check:

- destination ownership;
- destination write permission;
- archive ownership metadata;
- whether extraction is being run as the correct user.

Do not automatically use `sudo` without understanding the ownership consequences.

## Restored Files Have Wrong Ownership

Possible reasons:

- restore performed by a different user;
- filesystem does not support the same ownership model;
- archive created without required metadata;
- ownership restoration requires elevated privileges.

## Application Fails After Restore

Check:

- file ownership;
- directory permissions;
- configuration syntax;
- expected paths;
- database consistency;
- application version compatibility;
- SELinux or AppArmor controls;
- service logs.

A successful file extraction does not guarantee application recovery.

---

# Cleanup and Retention

Temporary restore directories should be removed after verification.

Backup archives should not be deleted until retention requirements are met.

A retention policy may define:

```text
daily backups: 7
weekly backups: 4
monthly backups: 12
```

The exact policy depends on:

- data change rate;
- recovery objectives;
- available storage;
- legal requirements;
- business impact.

---

# Final Verification Table

| Check | Result |
|---|---|
| Isolated source location prepared | Passed |
| Backup destination prepared | Passed |
| Restore test location prepared | Passed |
| Test data created | Passed |
| Source metadata inspected | Passed |
| Compressed archive created | Passed |
| Archive contents listed | Passed |
| Ownership metadata verified | Passed |
| Permission metadata verified | Passed |
| Gzip integrity test passed | Passed |
| SHA-256 checksum generated | Passed |
| Checksum verification passed | Passed |
| Source loss simulated | Passed |
| Source absence verified | Passed |
| Archive restored to isolated location | Passed |
| Restored structure verified | Passed |
| Ownership and permissions verified | Passed |
| Restored content verified | Passed |
| Archive content read directly | Passed |
| Byte-for-byte comparison passed | Passed |
| Original location restored | Passed |
| Original metadata verified | Passed |
| Temporary restore copy removed | Passed |
| Backup archive retained | Passed |

---

# Lessons Learned

- A backup file existing does not prove that it is usable.
- Archive contents should be inspected before restore.
- Compression integrity and content completeness are different checks.
- Checksums detect file changes and corruption.
- Checksums do not replace restore testing.
- Restore should preferably begin in an isolated location.
- Ownership, permissions, timestamps, and content must all be verified.
- `tar -C` helps produce clean relative paths.
- `tar -O` can read a file directly from an archive.
- `diff` can prove content equality.
- `cp -a` preserves metadata during the final copy.
- Backup and restore procedures must include cleanup and rollback planning.
- Local backups do not protect against complete host or storage failure.
- Snapshots and backups serve different purposes.
- Restore testing is the strongest evidence that a backup is operationally useful.

---

# Outcome

A complete and verified backup and restore workflow was successfully performed on `phoenix-linux-01`.

The final state includes:

```text
Restored source:
  /home/igor/phoenix-backup-lab

Verified backup:
  /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz

Checksum:
  /mnt/phoenix-data/backups/phoenix-backup-lab.tar.gz.sha256

Temporary restore copy:
  removed
```

The backup passed archive inspection, gzip integrity testing, SHA-256 verification, isolated restore testing, metadata validation, content validation, and byte-for-byte comparison.