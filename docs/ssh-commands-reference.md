# SSH Commands Reference

This document summarizes the SSH commands used during Project Phoenix Module 2.

---

## Connection

### Connect using the SSH alias

```bash
ssh lenovo2
```

Uses the local SSH client configuration stored in:

```text
~/.ssh/config
```

### Connect using an explicit private key

```bash
ssh -i ~/.ssh/id_ed25519_homelab igor@SERVER_IP
```

Useful when testing a specific key or troubleshooting the SSH client configuration.

### Exit the remote session

```bash
exit
```

Closes the current SSH session and returns to the local workstation.

---

## Session Verification

### Show the current user

```bash
whoami
```

Confirms which user account is active.

### Show the current hostname

```bash
hostname
```

Confirms which system is being administered.

### Show the current directory

```bash
pwd
```

Displays the current working directory.

### Show SSH connection details

```bash
echo $SSH_CONNECTION
```

Displays:

- client IP address;
- client source port;
- server IP address;
- server SSH port.

---

## SSH Client and Server Verification

### Show the SSH client version

```bash
ssh -V
```

Confirms that the OpenSSH client is installed.

### Check SSH server status

```bash
systemctl status ssh
```

Shows whether the SSH server is installed and running.

### Check the listening SSH port

```bash
sudo ss -ltnp | grep ':22'
```

Shows which process is listening on TCP port 22.

---

## SSH Configuration Inspection

### Show effective SSH server settings

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Displays the active SSH authentication settings after all configuration files have been processed.

---

## SSH Keys

### Create a dedicated HomeLab SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_homelab -C "igor@project-phoenix-homelab"
```

Creates:

```text
~/.ssh/id_ed25519_homelab
~/.ssh/id_ed25519_homelab.pub
```

The private key must never be shared.

### Copy the public key to Lenovo2

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub igor@SERVER_IP
```

Adds the public key to:

```text
/home/igor/.ssh/authorized_keys
```

### List keys loaded in the SSH agent

```bash
ssh-add -l
```

Displays the fingerprints of keys currently available to the SSH agent.

---

## Host Identity

### Find the stored Lenovo2 host key

```bash
ssh-keygen -F SERVER_IP
```

Searches the local `known_hosts` file.

### Show the stored ED25519 host fingerprint

```bash
ssh-keygen -F SERVER_IP | grep ssh-ed25519 | ssh-keygen -lf -
```

Displays the server fingerprint remembered by the workstation.

### Show the active Lenovo2 ED25519 fingerprint

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Displays the fingerprint currently used by Lenovo2.

The client and server fingerprints should match.

---

## File and Permission Inspection

### List file ownership and permissions

```bash
ls -l FILE_NAME
```

Shows:

- file type;
- permissions;
- owner;
- group;
- size;
- modification time.

### Change file ownership

```bash
sudo chown USER:GROUP FILE_NAME
```

Example:

```bash
sudo chown igor:igor permission-test.txt
```

Use `chown` when a file belongs to the wrong user or group.

### Read a file

```bash
cat FILE_NAME
```

Displays the file contents.

### Read a filename containing spaces

```bash
cat "notes copy.txt"
```

Quotes prevent the shell from treating the name as multiple arguments.

---

## Important Files

| File | Purpose |
|------|---------|
| `~/.ssh/config` | Local SSH client profiles and aliases |
| `~/.ssh/known_hosts` | Trusted SSH server identities |
| `~/.ssh/id_ed25519_homelab` | Private HomeLab SSH key |
| `~/.ssh/id_ed25519_homelab.pub` | Public HomeLab SSH key |
| `~/.ssh/authorized_keys` | Public keys allowed to access an account |
| `/etc/ssh/sshd_config` | Main SSH server configuration |
| `/etc/ssh/ssh_host_ed25519_key.pub` | Public ED25519 host key |
| `/etc/hosts` | Local hostname-to-address mappings |

---

## VS Code Remote SSH Checks

Confirm that the VS Code status bar displays:

```text
SSH: lenovo2
```

Confirm that the integrated terminal prompt shows:

```text
igor@lenovo2
```

These checks confirm that the editor and terminal are working on the remote server.

---

## Troubleshooting Pattern

Use this sequence:

1. define the symptom;
2. inspect the file or service;
3. confirm ownership, permissions, or status;
4. identify the root cause;
5. apply one controlled change;
6. verify the result.

---

## Best Practices

- Use SSH keys instead of passwords whenever possible.
- Protect private keys with a strong passphrase.
- Never commit private keys to Git.
- Verify the host fingerprint before trusting a server.
- Keep recovery access available before changing SSH settings.
- Prefer `chown` when ownership is wrong.
- Avoid unnecessary permission widening with `chmod`.
- Use sanitized placeholders such as `SERVER_IP` in public documentation.
- Verify whether the active shell is local or remote before running commands.

---

## Related Documents

- `runbooks/ssh-access-lenovo2.md`
- `README.md`