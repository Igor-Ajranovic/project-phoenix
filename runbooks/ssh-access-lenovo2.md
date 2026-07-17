# SSH Hardening — phoenix-linux-01

## Document Information

| Field | Value |
|---|---|
| Project | Project Phoenix |
| System | phoenix-linux-01 |
| Platform | Ubuntu Server 24.04.1 LTS |
| Environment | Proxmox HomeLab |
| Network | Servers network |
| Management IP | `192.168.20.101` |
| SSH user | `igor` |
| SSH port | `22` |
| Authentication method | Ed25519 public key |
| Status | Completed and verified |

## Purpose

This runbook documents the deployment, verification, and hardening of SSH access for the `phoenix-linux-01` lab virtual machine.

The objective was to establish secure remote administration using public-key authentication while disabling password-based and root SSH access.

## Safety and Recovery Controls

The following safety controls were used before changing SSH authentication:

- The virtual machine remained accessible through the Proxmox console.
- Two working SSH sessions were kept open during configuration changes.
- SSH configuration syntax was validated before every reload.
- The SSH service was reloaded instead of restarted.
- A new SSH connection was tested before existing sessions were closed.
- An Omada Controller configuration backup was created before adding the fixed IP reservation.

These controls reduced the risk of losing remote access.

## Network Configuration

The virtual machine received the following address:

```text
192.168.20.101/24
```

A fixed DHCP reservation was created in Omada Controller for the VM network interface.

This provides a stable management address while retaining centralized DHCP management.

## Initial SSH Verification

The following commands were used to confirm the system identity and SSH availability:

```bash
whoami
hostname
ssh -V
systemctl status ssh
sudo ss -ltnp | grep ':22'
```

Verified results:

- Logged-in user: `igor`
- Hostname: `phoenix-linux-01`
- SSH service: active and running
- Listening port: TCP `22`
- IPv4 listener: `0.0.0.0:22`
- IPv6 listener: `[::]:22`

> `ssh -V` displays the installed OpenSSH client version. The running SSH server was verified separately through systemd and the listening socket.

## Initial Effective SSH Configuration

The initial effective configuration was checked with:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Initial relevant settings:

```text
port 22
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication yes
```

This confirmed that public-key authentication was available, but password authentication was still enabled.

## Public-Key Deployment

An existing dedicated HomeLab Ed25519 key was used:

```text
~/.ssh/id_ed25519_homelab
```

The public key was installed from the administration laptop:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub igor@192.168.20.101
```

The private key was never copied to the server.

## Public-Key Authentication Test

Key-based access was tested explicitly:

```bash
ssh -i ~/.ssh/id_ed25519_homelab igor@192.168.20.101
```

The connection succeeded without requesting the remote account password.

The remote identity was verified with:

```bash
whoami
hostname
```

Expected and confirmed output:

```text
igor
phoenix-linux-01
```

## Authorized Keys Permissions

The server-side SSH directory and authorized key file were inspected:

```bash
ls -la ~/.ssh
cat ~/.ssh/authorized_keys
```

Verified permissions:

```text
~/.ssh             700
~/.ssh/authorized_keys 600
```

Verified ownership:

```text
igor:igor
```

The public-key content is intentionally excluded from this public documentation.

## SSH Client Alias

The following host entry was added to the administration laptop's local SSH configuration:

```ssh
Host phoenix-linux-01
    HostName 192.168.20.101
    User igor
    IdentityFile ~/.ssh/id_ed25519_homelab
```

The alias was tested with:

```bash
ssh phoenix-linux-01
```

The connection completed successfully.

## Key-Only Authentication Test

To confirm that SSH was not silently falling back to password authentication, the following client-side test was performed:

```bash
ssh -o PasswordAuthentication=no -o IdentitiesOnly=yes phoenix-linux-01
```

The connection succeeded, confirming that the configured Ed25519 key was sufficient.

## SSH Hardening Configuration

A dedicated configuration drop-in was created:

```text
/etc/ssh/sshd_config.d/10-phoenix-hardening.conf
```

File contents:

```ssh
# Project Phoenix SSH hardening
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

A separate drop-in file was used instead of editing the main `/etc/ssh/sshd_config` file.

This makes the configuration easier to audit, document, and reverse.

## Configuration Precedence Issue

The hardening file was initially named:

```text
99-phoenix-hardening.conf
```

However, the following cloud-init configuration was already present:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

It contained:

```ssh
PasswordAuthentication yes
```

The active definitions were located with:

```bash
sudo grep -RniE '^[[:space:]]*PasswordAuthentication' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

OpenSSH retained the first applicable value encountered during configuration parsing. Because `50-cloud-init.conf` was processed before `99-phoenix-hardening.conf`, password authentication remained enabled.

The Project Phoenix file was renamed so that it loads earlier:

```bash
sudo mv /etc/ssh/sshd_config.d/99-phoenix-hardening.conf \
  /etc/ssh/sshd_config.d/10-phoenix-hardening.conf
```

This corrected the precedence issue without modifying the cloud-init-managed file.

## Syntax Validation

The configuration was validated before applying it:

```bash
sudo sshd -t
```

No output was returned, indicating valid SSH server configuration syntax.

## Applying the Configuration

The SSH service configuration was reloaded:

```bash
sudo systemctl reload ssh
```

A reload was used instead of a restart to avoid unnecessarily interrupting active SSH sessions.

## Effective Configuration Verification

The active SSH configuration was verified again:

```bash
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication) '
```

Confirmed result:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

## Final Connection Test

A completely new SSH session was opened from the administration laptop:

```bash
ssh phoenix-linux-01
```

The connection succeeded using the configured Ed25519 key.

This confirmed that the server remained remotely accessible after hardening.

## Negative Password Authentication Test

A password-only connection was intentionally attempted:

```bash
ssh -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  igor@192.168.20.101
```

Expected and confirmed result:

```text
Permission denied (publickey).
```

This verified that password authentication was actually disabled and that the server accepted public-key authentication only.

## Final Security State

| Control | State |
|---|---|
| SSH service active | Yes |
| SSH port | `22` |
| Public-key authentication | Enabled |
| Password authentication | Disabled |
| Root SSH login | Disabled |
| SSH alias tested | Yes |
| New key-based session tested | Yes |
| Password-only login rejected | Yes |
| Proxmox console recovery available | Yes |
| Fixed management IP | Yes |

## Troubleshooting Notes

### Password authentication remained enabled

Symptom:

```text
passwordauthentication yes
```

remained active after adding a hardening drop-in.

Cause:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

was processed before the custom configuration and set:

```ssh
PasswordAuthentication yes
```

Resolution:

The custom configuration was renamed to:

```text
10-phoenix-hardening.conf
```

The syntax was validated, SSH was reloaded, and the effective configuration was checked again.

### SSH alias resolved to `127.0.1.1`

Running this command from inside the server:

```bash
ssh phoenix-linux-01
```

resolved the local hostname to:

```text
127.0.1.1
```

This attempted to connect from the server back to itself.

Resolution:

SSH client alias tests must be performed from the administration laptop where the relevant `~/.ssh/config` entry exists.

## Recovery Procedure

If key-based SSH access stops working:

1. Open the `phoenix-linux-01` console in Proxmox.
2. Log in with the local `igor` account.
3. Inspect the custom configuration:

   ```bash
   sudo cat /etc/ssh/sshd_config.d/10-phoenix-hardening.conf
   ```

4. Validate the configuration:

   ```bash
   sudo sshd -t
   ```

5. Temporarily move the custom file out of the SSH configuration directory if required:

   ```bash
   sudo mv /etc/ssh/sshd_config.d/10-phoenix-hardening.conf \
     /root/10-phoenix-hardening.conf.disabled
   ```

6. Reload SSH:

   ```bash
   sudo systemctl reload ssh
   ```

7. Diagnose the key, ownership, and permission problem before restoring hardened access.

## Security Considerations

- The private SSH key must remain only on trusted administration devices.
- Public keys may be installed on managed servers.
- Private keys, passwords, complete key material, and sensitive fingerprints must not be committed to Git.
- Proxmox console access remains the primary recovery method.
- Password authentication should not be re-enabled as a permanent workaround.
- SSH configuration changes must always be syntax-tested before reload or restart.

## Outcome

The `phoenix-linux-01` virtual machine now supports secure key-based remote administration.

The completed implementation includes:

- stable network addressing;
- a dedicated SSH client alias;
- Ed25519 public-key authentication;
- disabled password authentication;
- disabled root SSH access;
- tested configuration precedence;
- verified positive and negative authentication tests;
- documented console recovery procedures.

This lab demonstrates a controlled SSH hardening workflow suitable for Linux infrastructure administration.