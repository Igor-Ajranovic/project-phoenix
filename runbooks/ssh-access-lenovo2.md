# SSH Access Runbook — Lenovo2

## Purpose

This runbook documents the standard procedure for securely accessing and administering the Lenovo2 HomeLab server.

Lenovo2 is a production-like HomeLab service node. Changes should be planned, verified, and documented before execution.

---

## Target Server

| Item | Value |
|------|------|
| Host Alias | `lenovo2` |
| Hostname | `lenovo2` |
| Administrator User | `igor` |
| Network Address | `SERVER_IP` |
| Primary Authentication | SSH Key Authentication |
| Temporary Fallback | Password Authentication |
| Remote Editor | Visual Studio Code Remote - SSH |

---

## Standard Terminal Connection

Connect using the local SSH alias:

```bash
ssh lenovo2
```

The alias is defined in the local SSH client configuration:

```text
~/.ssh/config
```

Example configuration:

```ssh
Host lenovo2
    HostName SERVER_IP
    User igor
    IdentityFile ~/.ssh/id_ed25519_homelab
```

---

## Explicit Key Connection

Use the following command when testing a specific SSH key or troubleshooting the SSH client configuration:

```bash
ssh -i ~/.ssh/id_ed25519_homelab igor@SERVER_IP
```

---

## Session Verification

After connecting, verify the current user and target hostname:

```bash
whoami
hostname
```

Expected output:

```text
igor
lenovo2
```

Display the active SSH connection details:

```bash
echo $SSH_CONNECTION
```

The output contains:

1. client IP address;
2. client source port;
3. server IP address;
4. server SSH port.

---

## SSH Service Verification

Check whether the SSH server is running:

```bash
systemctl status ssh
```

Expected service state:

```text
Active: active (running)
```

Check which process is listening on the SSH port:

```bash
sudo ss -ltnp | grep ':22'
```

Expected result:

```text
LISTEN ... 0.0.0.0:22 ...
LISTEN ... [::]:22 ...
```

This indicates that the SSH server is listening on IPv4 and IPv6 interfaces.

---

## Effective SSH Authentication Settings

Display the active SSH server settings after all configuration files have been processed:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Verified baseline:

```text
port 22
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication yes
```

### Interpretation

- SSH uses the standard TCP port `22`.
- Public key authentication is enabled.
- Password authentication is currently enabled as a fallback.
- Direct root login using a password is not permitted.

Do not disable password authentication until SSH key access and recovery options have been fully verified.

---

## SSH Key Authentication

The administrator workstation uses a dedicated HomeLab SSH key:

```text
~/.ssh/id_ed25519_homelab
```

The corresponding public key is stored in:

```text
~/.ssh/id_ed25519_homelab.pub
```

The public key is installed on Lenovo2 in:

```text
/home/igor/.ssh/authorized_keys
```

The private key must remain only on trusted administrator devices and must never be shared or committed to Git.

---

## SSH Agent Verification

List the SSH keys currently loaded into the local SSH agent:

```bash
ssh-add -l
```

The SSH agent temporarily keeps unlocked keys available in memory. This allows repeated SSH connections without entering the key passphrase every time during the same desktop session.

---

## Host Identity Verification

The workstation stores trusted SSH server identities in:

```text
~/.ssh/known_hosts
```

Find the saved Lenovo2 host key:

```bash
ssh-keygen -F SERVER_IP
```

Display the locally stored ED25519 fingerprint:

```bash
ssh-keygen -F SERVER_IP | grep ssh-ed25519 | ssh-keygen -lf -
```

Display the active ED25519 host fingerprint directly on Lenovo2:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

The client and server fingerprints must match.

An unexpected fingerprint change must not be accepted without investigation. Legitimate causes may include a server reinstall or regenerated SSH host keys.

---

## Visual Studio Code Remote Access

Lenovo2 can also be administered through the Visual Studio Code Remote - SSH extension.

### Connection Procedure

1. Open **Remote Explorer** in Visual Studio Code.
2. Select the `lenovo2` SSH target.
3. Connect using the existing SSH client profile.
4. Confirm that the lower-left status bar displays:

```text
SSH: lenovo2
```

5. Open the integrated terminal.
6. Confirm that the terminal prompt displays:

```text
igor@lenovo2
```

These checks confirm that both the editor and terminal are operating on the remote server.

### Safe Working Area

For training and non-destructive testing, use:

```text
/home/igor/linux-praksa
```

Avoid modifying production service files, SSH configuration, CasaOS files, Docker configuration, DNS settings, or reverse proxy configuration without a tested plan.

---

## Remote Editing Verification

A remote file can be opened and edited through Visual Studio Code.

After saving the file, verify the result in the integrated terminal:

```bash
cat FILE_NAME
```

This confirms that the change was written to Lenovo2 rather than the local workstation.

Avoid unnecessary spaces in server-side filenames. Prefer:

```text
notes-copy.txt
```

instead of:

```text
notes copy.txt
```

When a filename contains spaces, quote it:

```bash
cat "notes copy.txt"
```

---

## Permission Troubleshooting

### Symptom

Visual Studio Code displays an error similar to:

```text
EACCES: permission denied
```

### Investigation

Inspect the file ownership and permissions:

```bash
ls -l permission-test.txt
```

Example problematic result:

```text
-rw-r--r-- 1 root root ... permission-test.txt
```

Interpretation:

- owner `root`: read and write;
- group `root`: read only;
- other users, including `igor`: read only.

### Root Cause

The file was created with elevated privileges and belongs to `root`, while it is intended to be managed by the normal administrator account.

### Resolution

Correct the ownership:

```bash
sudo chown igor:igor permission-test.txt
```

Verify the result:

```bash
ls -l permission-test.txt
```

Expected ownership:

```text
-rw-r--r-- 1 igor igor ... permission-test.txt
```

### Verification

Save the file again through Visual Studio Code and verify its contents:

```bash
cat permission-test.txt
```

### Engineering Lesson

Use `chown` when ownership is incorrect.

Do not use `chmod` merely to give wider permissions to a file that belongs to the wrong user.

---

## System File Privilege Boundary

System configuration files such as:

```text
/etc/hosts
```

may be readable by normal users but writable only by `root`.

Example permissions:

```text
-rw-r--r-- 1 root root ... /etc/hosts
```

This means:

- owner `root`: read and write;
- group `root`: read only;
- other users: read only.

Visual Studio Code may allow the file to be opened and edited in memory, but Linux permissions are enforced when saving.

Do not test write access on production system files without a valid change request, backup, validation procedure, and recovery plan.

---

## Recovery Options

Available recovery methods include:

- physical access using a keyboard and monitor;
- password authentication while it remains enabled;
- verified SSH key authentication;
- Tailscale as an alternative network path;
- an existing active SSH session during planned configuration changes.

Lenovo2 is a physical mini PC and does not have a Proxmox console.

---

## Safety Procedure Before SSH Configuration Changes

Before changing SSH server settings:

1. confirm physical recovery access;
2. keep the current SSH session open;
3. open a second test session;
4. back up the current SSH configuration;
5. validate the configuration with `sshd -t`;
6. change one setting at a time;
7. verify access before closing the original session;
8. document the change and rollback procedure.

---

## Security Notes

- Never share private SSH keys.
- Protect private keys with a strong passphrase.
- Store private keys only on trusted administrator devices.
- Never commit private keys, passwords, tokens, or sensitive configuration to Git.
- Use separate keys for GitHub and HomeLab administration.
- Verify unexpected SSH host fingerprint changes.
- Keep a recovery method available before hardening SSH.
- Do not expose SSH directly to the public internet without a reviewed security model.
- Use sanitized placeholders such as `SERVER_IP` in public documentation.

---

## Lessons Learned

- SSH clients and SSH servers have different roles.
- Password login and key authentication are separate authentication methods.
- A user key proves the administrator's identity to the server.
- A host key proves the server's identity to the client.
- `authorized_keys` controls which public keys may access an account.
- `known_hosts` stores trusted server identities.
- SSH aliases simplify repeated administration.
- VS Code Remote - SSH does not bypass Linux permissions.
- Ownership should be corrected before widening permissions.
- Always confirm whether work is being performed locally or remotely.
- Troubleshooting should follow: symptom, evidence, root cause, resolution, verification.

---

## Current Status

- SSH client verified on the Ubuntu administration laptop.
- SSH server verified on Lenovo2.
- Password authentication verified.
- Dedicated HomeLab SSH key created.
- Public key installed on Lenovo2.
- SSH key authentication verified.
- SSH agent verified.
- SSH alias `lenovo2` configured.
- Host fingerprint verified.
- Visual Studio Code Remote - SSH verified.
- Remote file editing verified.
- Permission troubleshooting completed.
- Production SSH hardening not yet performed.

---

## Related Documents

- `docs/ssh-commands-reference.md`
- `README.md`