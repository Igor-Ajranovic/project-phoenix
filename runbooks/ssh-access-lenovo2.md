# SSH Access Runbook – Lenovo2

## Purpose

This runbook documents the standard procedure for securely accessing the Lenovo2 HomeLab server.

---

## Target Server

| Item | Value |
|------|------|
| Host Alias | `lenovo2` |
| Hostname | `lenovo2` |
| User | `igor` |
| SSH Key Authentication |
| Fallback | Password Authentication |

---

## Standard Connection

```bash
ssh lenovo2
```

## Verification

After connecting, run:

```bash
whoami
hostname
```

## Expected output

```text
igor
lenovo2
```

## Recovery Options

- Physical access (keyboard and monitor)
- Password Authentication (temporary fallback)
- SSH key authentication
- Tailscale remote access


## Security Notes
- Never share the private SSH key.
- Keep the private key protected with a passphrase.
- Verify the SSH host fingerprint before accepting unexpected key changes.
- Do not disable password authentication until SSH key authentication has been fully verified.
- Store private keys only on trusted administrator devices.


## Status

✅ SSH key authentication verified

✅ Host fingerprint verified

✅ SSH alias configured