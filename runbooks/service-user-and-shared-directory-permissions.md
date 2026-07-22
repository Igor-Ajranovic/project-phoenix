# Service User and Shared Directory Permissions

## Learning Context

| Field | Value |
|---|---|
| Module | 2 — Linux Administration |
| Topic | Permissions in Real Service Scenarios |
| Subtopic | Service Accounts, Shared Groups, setgid, umask, and Least Privilege |
| Project | Project Phoenix |
| System | `phoenix-linux-01` |
| Platform | Ubuntu Server 24.04.1 LTS |
| Storage | `/mnt/phoenix-data` |
| Status | Completed and verified |

## Purpose

This runbook documents a practical Linux permissions lab using a dedicated service account and a shared service-data directory.

The objectives were to:

- avoid running a service as `root`;
- create a non-login system user;
- provide a dedicated data directory for the service;
- allow the service account to write to its own storage;
- allow the administrator to access the same directory through group membership;
- preserve a consistent shared group on new files and subdirectories;
- verify that unauthorized users cannot write to the directory;
- apply the principle of least privilege.

## Final Access Model

```text
Service account: phoenixsvc
Administrator: igor
Shared group: phoenixsvc
Service directory: /mnt/phoenix-data/service-data
Directory mode: 2775
```

Result:

```text
phoenixsvc → read and write
igor       → read and write through group membership
nobody     → no write access
```

---

# Part 1 — Initial Directory State

The persistent data mount was:

```text
/mnt/phoenix-data
```

Its initial ownership and permissions were checked with:

```bash
ls -ld /mnt/phoenix-data
```

Observed state:

```text
drwxr-xr-x 3 igor igor ... /mnt/phoenix-data
```

This meant:

- owner: `igor`;
- group: `igor`;
- owner could write;
- group and other users could read and enter;
- the directory was tied to a personal user rather than a dedicated service identity.

For a service scenario, the data should belong to a dedicated service account rather than a personal administrator account.

---

# Part 2 — Dedicated Service Account

A system user was created:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  phoenixsvc
```

## Option Explanation

```text
--system
```

Creates a system account with a system UID.

```text
--no-create-home
```

Does not create a home directory.

```text
--shell /usr/sbin/nologin
```

Prevents normal interactive login.

```text
phoenixsvc
```

Defines the service account name.

## Account Verification

The account was checked with:

```bash
getent passwd phoenixsvc
id phoenixsvc
```

Confirmed result:

```text
UID: 999
Primary group: phoenixsvc
Shell: /usr/sbin/nologin
```

The passwd entry contained a home path:

```text
/home/phoenixsvc
```

However, the directory was not created because `--no-create-home` was used.

This was verified with:

```bash
ls -ld /home/phoenixsvc
```

Result:

```text
No such file or directory
```

The administrator also verified that a single command could be executed as the service account:

```bash
sudo -u phoenixsvc whoami
```

Result:

```text
phoenixsvc
```

This does not enable interactive login.

It only allows the administrator to execute a controlled command as that identity.

---

# Part 3 — Dedicated Service Data Directory

A dedicated service directory was created:

```bash
sudo mkdir -p /mnt/phoenix-data/service-data
```

Ownership was assigned to the service account:

```bash
sudo chown phoenixsvc:phoenixsvc \
  /mnt/phoenix-data/service-data
```

The result was verified:

```bash
ls -ld /mnt/phoenix-data/service-data
```

Observed state:

```text
drwxr-xr-x phoenixsvc phoenixsvc /mnt/phoenix-data/service-data
```

## Service Write Test

The service account created a test file:

```bash
sudo -u phoenixsvc \
  touch /mnt/phoenix-data/service-data/service-test.txt
```

The file was verified:

```bash
ls -l /mnt/phoenix-data/service-data
```

Confirmed result:

```text
-rw-rw-r-- 1 phoenixsvc phoenixsvc ... service-test.txt
```

This proved that the service account could write to its dedicated storage.

---

# Part 4 — Initial Administrator Access Test

The administrator attempted to create a file:

```bash
touch /mnt/phoenix-data/service-data/igor-test.txt
```

Result:

```text
Permission denied
```

This was expected because:

- the directory owner was `phoenixsvc`;
- the group was `phoenixsvc`;
- the directory mode was `755`;
- group write permission was not enabled;
- `igor` was not yet a member of the `phoenixsvc` group.

---

# Part 5 — Shared Group Membership

The administrator was added to the service group:

```bash
sudo usermod -aG phoenixsvc igor
```

## Important Option

```text
-aG
```

Means:

- `-G` adds supplementary group membership;
- `-a` appends the group without replacing existing supplementary groups.

Omitting `-a` could remove the user from existing supplementary groups.

## Membership Verification

The group was checked with:

```bash
getent group phoenixsvc
id igor
```

Confirmed result:

```text
phoenixsvc:x:988:igor
```

and:

```text
igor ... groups=...,988(phoenixsvc)
```

A new SSH session was opened so the login process would include the new supplementary group.

The administrator then retried the write test.

The operation still failed.

This showed that group membership alone is not sufficient if the directory does not grant group write permission.

---

# Part 6 — Group Write Permission

The directory mode was changed:

```bash
sudo chmod 775 /mnt/phoenix-data/service-data
```

The result was checked:

```bash
ls -ld /mnt/phoenix-data/service-data
```

Confirmed state:

```text
drwxrwxr-x phoenixsvc phoenixsvc /mnt/phoenix-data/service-data
```

Permission breakdown:

```text
owner: rwx
group: rwx
other: r-x
```

The administrator retried:

```bash
touch /mnt/phoenix-data/service-data/igor-test.txt
```

The command succeeded.

## File Ownership Result

The created file was:

```text
-rw-rw-r-- 1 igor igor ... igor-test.txt
```

This showed that:

- the file owner was the user who created it;
- the file group was the creator's primary group;
- directory group ownership was not automatically inherited.

This behavior is not ideal for a shared service directory.

---

# Part 7 — setgid on the Directory

The setgid bit was added:

```bash
sudo chmod 2775 /mnt/phoenix-data/service-data
```

The result was verified:

```bash
ls -ld /mnt/phoenix-data/service-data
```

Confirmed state:

```text
drwxrwsr-x phoenixsvc phoenixsvc /mnt/phoenix-data/service-data
```

The `s` in the group execute position indicates setgid:

```text
rws
```

## setgid Behavior on Directories

When setgid is applied to a directory:

- new files inherit the directory's group;
- new subdirectories inherit the directory's group;
- new subdirectories also inherit the setgid bit.

The file owner remains the user or service that created the file.

## setgid File Test

The administrator created:

```bash
touch \
  /mnt/phoenix-data/service-data/igor-setgid-test.txt
```

Result:

```text
owner: igor
group: phoenixsvc
```

This confirmed that setgid group inheritance worked.

Comparison:

```text
Before setgid:
igor-test.txt → igor:igor
```

```text
After setgid:
igor-setgid-test.txt → igor:phoenixsvc
```

---

# Part 8 — umask

The administrator's current umask was checked:

```bash
umask
```

Confirmed result:

```text
0002
```

## File Permission Calculation

Maximum normal file permissions:

```text
666 → rw-rw-rw-
```

Apply umask:

```text
666 - 002 = 664
```

Result:

```text
rw-rw-r--
```

## Directory Permission Calculation

Maximum normal directory permissions:

```text
777 → rwxrwxrwx
```

Apply umask:

```text
777 - 002 = 775
```

Result:

```text
rwxrwxr-x
```

## Combined Behavior

The shared directory model relies on two separate mechanisms:

```text
setgid
```

controls the inherited group.

```text
umask 0002
```

preserves group write permission.

Combined result:

```text
new file:
owner = creator
group = phoenixsvc
mode = 664
```

```text
new directory:
owner = creator
group = phoenixsvc
mode = 2775
```

---

# Part 9 — Subdirectory Inheritance

The administrator created a subdirectory:

```bash
mkdir \
  /mnt/phoenix-data/service-data/igor-directory
```

The result was verified:

```bash
ls -ld \
  /mnt/phoenix-data/service-data/igor-directory
```

Confirmed result:

```text
drwxrwsr-x igor phoenixsvc igor-directory
```

This showed that the new subdirectory inherited:

- group `phoenixsvc`;
- group write permission;
- the setgid bit.

## Cross-Account Write Test

The service account created a file inside the administrator-created directory:

```bash
sudo -u phoenixsvc \
  touch \
  /mnt/phoenix-data/service-data/igor-directory/service-write-test.txt
```

The file was verified:

```bash
ls -l \
  /mnt/phoenix-data/service-data/igor-directory
```

Confirmed result:

```text
-rw-rw-r-- phoenixsvc phoenixsvc service-write-test.txt
```

This proved that the administrator and service account could safely share the same directory tree.

---

# Part 10 — Negative Access Test

The user `nobody` was used to represent an account outside the service group.

The write test was:

```bash
sudo -u nobody \
  touch /mnt/phoenix-data/service-data/nobody-test.txt
```

Result:

```text
Permission denied
```

This confirmed the intended least-privilege model:

```text
service account → allowed
authorized administrator → allowed
unrelated account → denied
```

---

# Part 11 — Cleanup

Temporary lab files were removed:

```bash
rm \
  /mnt/phoenix-data/service-data/igor-test.txt \
  /mnt/phoenix-data/service-data/igor-setgid-test.txt
```

The nested test file and directory were removed:

```bash
sudo rm \
  /mnt/phoenix-data/service-data/igor-directory/service-write-test.txt

rmdir \
  /mnt/phoenix-data/service-data/igor-directory
```

The original service-created file was retained as evidence:

```text
service-test.txt
```

## Final Verification

The final directory state was checked:

```bash
ls -ld /mnt/phoenix-data/service-data
ls -l /mnt/phoenix-data/service-data
```

Confirmed result:

```text
drwxrwsr-x 2 phoenixsvc phoenixsvc ... service-data
```

and:

```text
-rw-rw-r-- 1 phoenixsvc phoenixsvc ... service-test.txt
```

---

# Final Permissions Model

```text
/mnt/phoenix-data/service-data
owner: phoenixsvc
group: phoenixsvc
mode: 2775
```

Access behavior:

| Identity | Access |
|---|---|
| `phoenixsvc` | Read, write, enter |
| `igor` | Read, write, enter through group |
| `nobody` | Read and enter only; no write |
| New files | Inherit group `phoenixsvc` |
| New directories | Inherit group `phoenixsvc` and setgid |

---

# Key Concepts

## Service Account

A dedicated account used by an application or service.

It should not normally have:

- an interactive shell;
- a personal home directory;
- unnecessary group membership;
- unrestricted sudo access.

## Owner

The individual user associated with a file or directory.

## Group

The shared access category associated with a file or directory.

## Supplementary Group

An additional group assigned to a user.

This allows controlled shared access without changing the user's primary group.

## setgid Directory

A directory where new content inherits the directory's group.

## umask

A mask that removes permissions from the maximum defaults when new files and directories are created.

## Least Privilege

Only the users and services that require access should receive it.

---

# Practical Applications

This model can be reused for:

- Docker bind-mount directories;
- web application data;
- database backup directories;
- shared configuration storage;
- log directories;
- monitoring-agent data;
- reverse-proxy certificate storage;
- application upload directories;
- backup and restore locations;
- service-generated reports.

Example:

```text
application service user
        +
administration group
        +
setgid directory
        +
group-friendly umask
        =
controlled shared access
```

---

# Operational Notes

- Do not run applications as `root` unless absolutely required.
- Use dedicated service accounts.
- Grant administrators access through groups.
- Group membership changes normally require a new login session.
- Group membership alone does not grant write access.
- The directory must also have group write permission.
- setgid controls group inheritance.
- umask controls default permissions.
- setgid does not change file ownership.
- Existing files are not automatically updated when setgid is added.
- Unauthorized users should be tested explicitly.
- Shared directories should be verified after reboot and application deployment.

---

# Recovery and Reversal

## Remove Administrator Group Access

```bash
sudo gpasswd -d igor phoenixsvc
```

The user must start a new login session before the removed membership is fully reflected.

## Remove setgid

```bash
sudo chmod 775 /mnt/phoenix-data/service-data
```

## Restore Service-Only Access

```bash
sudo chmod 755 /mnt/phoenix-data/service-data
```

## Remove the Service Account

Only after stopping and removing every dependent service:

```bash
sudo userdel phoenixsvc
```

Do not remove the account while files, systemd services, containers, or applications still depend on its UID or group.

---

# Final Verification Table

| Check | Result |
|---|---|
| System service account created | Passed |
| Interactive login blocked | Passed |
| Home directory absent | Passed |
| Dedicated service directory created | Passed |
| Service ownership assigned | Passed |
| Service write access verified | Passed |
| Administrator initially denied | Passed |
| Administrator added to service group | Passed |
| New login session tested | Passed |
| Group write permission enabled | Passed |
| Administrator write access verified | Passed |
| setgid applied | Passed |
| Group inheritance verified | Passed |
| umask reviewed | Passed |
| Subdirectory inheritance verified | Passed |
| Cross-account write test passed | Passed |
| Unauthorized-user write test rejected | Passed |
| Temporary test files removed | Passed |
| Final permissions verified | Passed |

## Lessons Learned

- A service should use a dedicated identity rather than a personal user account.
- Ownership and permissions solve different parts of access control.
- Group membership does not grant write access unless the directory permits it.
- New group membership may require a fresh login session.
- New files normally use the creator's primary group.
- setgid forces group inheritance inside shared directories.
- umask determines whether group write permission is preserved.
- A setgid directory with umask `0002` is a practical shared-workflow pattern.
- Positive and negative access tests are both required.
- Least privilege should be verified rather than assumed.

## Outcome

The `phoenix-linux-01` VM now has a controlled service-data access model.

The service account `phoenixsvc` owns the data directory.

The administrator `igor` can manage the same data through supplementary group membership.

New files and subdirectories inherit the shared service group, while unauthorized users remain unable to write.