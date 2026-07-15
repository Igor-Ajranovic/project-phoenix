# SSH Commands Reference

This document summarizes the SSH commands used during **Project Phoenix – Module 2: SSH Basics and Safe Access Verification**.

---

# Connection

## Connect using the SSH alias

```bash
ssh lenovo2
```

Uses the local SSH client configuration stored in:

```text
~/.ssh/config
```

---

## Connect using an explicit private key

```bash
ssh -i ~/.ssh/id_ed25519_homelab igor@SERVER_IP
```

Useful when testing a specific SSH key or troubleshooting SSH configuration.

---

# Session Verification

## Show the current user

```bash
whoami
```

Confirms which user account is currently active.

---

## Show the current hostname

```bash
hostname
```

Confirms which server is currently being administered.

---

## Show SSH connection details

```bash
echo $SSH_CONNECTION
```

Displays:

- Client IP address
- Client source port
- Server IP address
- Server SSH port

---

# SSH Service Verification

## Check SSH service status

```bash
systemctl status ssh
```

Shows whether the SSH server is installed and running.

---

## Check the listening SSH port

```bash
sudo ss -ltnp | grep ':22'
```

Shows which process is listening on TCP port 22.

---

# SSH Configuration Inspection

## Show effective SSH server settings

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Displays the active SSH authentication settings after all configuration files have been processed.

---

# SSH Keys

## Create a dedicated HomeLab SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_homelab -C "igor@project-phoenix-homelab"
```

Creates the following files:

```text
~/.ssh/id_ed25519_homelab
~/.ssh/id_ed25519_homelab.pub
```

> The private key must never be shared.

---

## Copy the public key to Lenovo2

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub igor@SERVER_IP
```

Adds the public key to:

```text
/home/igor/.ssh/authorized_keys
```

---

## List keys loaded in the SSH agent

```bash
ssh-add -l
```

Displays the fingerprints of all SSH keys currently loaded into the SSH agent.

---

# Host Identity Verification

## Find the stored host key

```bash
ssh-keygen -F SERVER_IP
```

Searches the local `known_hosts` file for the server.

---

## Display the stored ED25519 host fingerprint

```bash
ssh-keygen -F SERVER_IP | grep ssh-ed25519 | ssh-keygen -lf -
```

Displays the host fingerprint remembered by the local workstation.

---

## Display the Lenovo2 ED25519 host fingerprint

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Displays the fingerprint currently used by the SSH server.

The client and server fingerprints should always match.

---

# Important Files

| File | Purpose |
|------|---------|
| `~/.ssh/config` | Local SSH client configuration and host aliases |
| `~/.ssh/known_hosts` | Stores trusted SSH server identities |
| `~/.ssh/id_ed25519_homelab` | Private HomeLab SSH key |
| `~/.ssh/id_ed25519_homelab.pub` | Public HomeLab SSH key |
| `~/.ssh/authorized_keys` | Public keys allowed to authenticate |
| `/etc/ssh/sshd_config` | Main SSH server configuration |

---

# Best Practices

- Use SSH key authentication whenever possible.
- Protect private keys with a strong passphrase.
- Never commit private keys to Git.
- Never share private keys.
- Verify the SSH host fingerprint before trusting a server.
- Keep password authentication available until SSH key authentication has been fully verified.
- Always ensure an alternative recovery method exists before modifying SSH configuration.
- Test one SSH configuration change at a time.

---

# Related Documents

- `runbooks/ssh-access-lenovo2.md`
- `README.md`