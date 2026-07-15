# SSH Commands Reference

## Connection

### Connect to Lenovo2 using the SSH alias

```bash
ssh lenovo2
```

Uses the local SSH client configuration from
```bash
~/.ssh/config
```

## Connect using an explicit private key
```bash
ssh -i ~/.ssh/id_ed25519_homelab igor@192.168.20.102
```
- Useful when testing a specific SSH key.

## Session Verification
```bash
Show the current user
whoami
```
- Confirms which user account is active.

## Show the current hostname
```bash
hostname
```
- Confirms which system is currently being administered.


## Show SSH connection details
```bash
echo $SSH_CONNECTION
```
Displays:

- client IP address
- client source port
- server IP address
- server SSH port

## SSH Service Verification
- Check SSH service status
```bash
systemctl status ssh
```
- Shows whether the SSH server is installed and running.

## Check the listening SSH port
```bash
sudo ss -ltnp | grep ':22'
```
- Shows which process is listening on TCP port 22.

## SSH Configuration Inspection
- Show effective SSH server settings
```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication) '
```
- Displays the active SSH authentication settings after all configuration files are processed.

## SSH Keys
- Create a dedicated HomeLab SSH key
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_homelab -C "igor@project-phoenix-homelab"
```
- Creates:

- ~/.ssh/id_ed25519_homelab
- ~/.ssh/id_ed25519_homelab.pub

- The private key must never be shared.

## Copy the public key to Lenovo2
```bash
ssh-copy-id -i ~/.ssh/id_ed25519_homelab.pub igor@192.168.20.102
```
- Adds the public key to:

/home/igor/.ssh/authorized_keys

## List keys loaded in the SSH agent
```bash
ssh-add -l
```
- Displays key fingerprints currently available to the SSH agent.

## Host Identity
- Find the stored Lenovo2 host key
```bash
ssh-keygen -F 192.168.20.102
```
- Searches the local known_hosts file.

### Show the stored ED25519 host fingerprint
```bash
ssh-keygen -F 192.168.20.102 | grep ssh-ed25519 | ssh-keygen -lf -
```
- Displays the fingerprint remembered by the laptop.

## Show the Lenovo2 ED25519 host fingerprint
```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```
- Displays the fingerprint currently used by Lenovo2.
- The client and server fingerprints should match.

## Important Files
 | File	 |    Purpose  |
 | ------------- |---------------- |
 | ~/.ssh/config	| Local SSH client profiles and aliases |

 | ~/.ssh/known_hosts |	Stored server host identities |

 | ~/.ssh/id_ed25519_homelab |	Private HomeLab SSH key |

 | ~/.ssh/id_ed25519_homelab.pub	|  Public HomeLab SSH key |

| ~/.ssh/authorized_keys	|  Public keys allowed to access an account |

| /etc/ssh/sshd_config	|  Main SSH server configuration |