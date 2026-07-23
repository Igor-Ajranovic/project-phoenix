# Project Phoenix

Project Phoenix is a structured career transition program focused on Linux administration, infrastructure engineering, DevOps, automation, networking, containers, monitoring, cloud technologies, and professional technical documentation.

This repository documents practical engineering work completed in a real HomeLab environment. It serves as both a technical knowledge base and a public portfolio demonstrating the development of infrastructure engineering skills, operational discipline, troubleshooting ability, and documentation quality.

---

## Career Target

**Linux Infrastructure & DevOps Engineer**

Project Phoenix is designed to build practical readiness for roles in:

- Linux system administration;
- infrastructure operations;
- platform operations;
- DevOps engineering;
- junior infrastructure engineering;
- technical support engineering;
- systems administration;
- cloud and container operations.

---

## Core Principles

- Learn by building.
- Understand before automating.
- Document every meaningful change.
- Use real-world troubleshooting scenarios.
- Keep infrastructure reproducible.
- Treat security, backup, and recovery as core requirements.
- Use dedicated lab systems for risky or destructive exercises.
- Avoid unnecessary changes to production services.
- Build a public-quality technical portfolio.

---

## Learning Method

Project Phoenix follows an engineering-residency model.

Each learning period includes:

- one engineering topic;
- one practical lab;
- one troubleshooting scenario;
- one documentation task;
- one portfolio improvement.

The program prioritizes practical understanding before automation.

Graphical tools are used first when they teach the same concept effectively. Terminal commands are introduced when the command line is required for administration, troubleshooting, automation, or understanding the underlying system behavior.

All meaningful work is documented in professional English.

---

## Current Status

```text
Module 1 — Linux Foundations: Completed
Module 2 — Linux Administration: Completed
Module 3 — Git & Technical Documentation: In progress
```

---

## Current Focus

### Module 3 — Git & Technical Documentation

The current module focuses on:

- professional Git workflows;
- repository organization;
- Markdown documentation standards;
- technical writing;
- commit quality;
- version history;
- documentation review;
- portfolio presentation;
- reproducible documentation practices.

---

## Completed Modules

### Module 1 — Linux Foundations

Linux Foundations covered the essential concepts required to begin administering Linux systems safely.

Topics included:

- Linux system architecture;
- filesystem navigation;
- file and directory management;
- reading files and logs;
- text editing;
- users and groups;
- file ownership;
- permissions;
- package management basics;
- processes;
- systemd services;
- networking fundamentals;
- DNS;
- listening ports;
- SSH basics.

These foundations continue to be reinforced through later troubleshooting and engineering labs.

### Module 2 — Linux Administration

Module 2 expanded the foundations into practical Linux administration and operational engineering.

Completed topics include:

- SSH security and professional remote administration;
- system health analysis;
- failed-service troubleshooting;
- systemd services and timers;
- users, groups, and least-privilege access;
- shared directory permissions;
- setgid behavior;
- umask configuration;
- logging and log rotation;
- scheduled tasks and automation;
- backup and restore fundamentals;
- filesystem capacity monitoring;
- inode monitoring;
- process limits;
- resource control;
- storage administration;
- LVM;
- persistent mounts;
- boot analysis;
- kernel and initramfs inspection;
- rescue and emergency targets;
- safe `/etc/fstab` validation;
- package management;
- repository safety;
- unattended security updates;
- safe package cleanup;
- package audit logs.

---

## Primary Lab Environment

The primary Project Phoenix Linux administration system is:

```text
Hostname: phoenix-linux-01
Platform: Proxmox VE
Virtual Machine ID: 120
Operating System: Ubuntu Server 24.04 LTS
Primary User: igor
Purpose: Dedicated Linux administration and infrastructure engineering lab
```

### Storage

```text
Root filesystem:
  LVM logical volume
  ext4
  approximately 30 GB

Additional lab disk:
  Device: /dev/sdb1
  Filesystem: ext4
  Label: phoenix-data
  Mount point: /mnt/phoenix-data
```

### Remote Administration

```text
SSH alias: phoenix-linux-01
Authentication: Ed25519 SSH key
Recovery channel: Proxmox virtual console
```

The VM is used for practical Linux administration exercises without creating unnecessary risk for production HomeLab services.

---

## Supporting HomeLab Environment

Project Phoenix uses a real HomeLab as an engineering laboratory.

Technologies and platforms include:

- Proxmox VE;
- Ubuntu Server;
- Ubuntu Desktop;
- Docker;
- CasaOS;
- Caddy;
- AdGuard Home;
- Tailscale;
- TP-Link Omada;
- Synology NAS;
- Visual Studio Code;
- Git;
- GitHub;
- Kasm;
- n8n;
- Portainer;
- QEMU Guest Agent.

The HomeLab provides realistic infrastructure scenarios involving:

- networking;
- virtualization;
- DNS;
- reverse proxying;
- containers;
- storage;
- remote access;
- service monitoring;
- backup;
- recovery;
- troubleshooting.

---

## Repository Structure

```text
project-phoenix/
├── assets/
├── diagrams/
├── docs/
├── lab/
├── runbooks/
├── scripts/
├── .gitignore
└── README.md
```

### Directory Purpose

| Directory | Purpose |
|---|---|
| `assets/` | Images, screenshots, and supporting media |
| `diagrams/` | Architecture, network, service, and infrastructure diagrams |
| `docs/` | Knowledge base, reference material, and structured documentation |
| `lab/` | Practical exercises, experiments, and lab records |
| `runbooks/` | Operational procedures, recovery guides, and troubleshooting workflows |
| `scripts/` | Administrative and automation scripts |
| `.gitignore` | Files and directories excluded from version control |
| `README.md` | Main repository overview and portfolio entry point |

---

## Documentation Standards

All technical deliverables are written in professional English.

This includes:

- README files;
- runbooks;
- diagrams;
- Git commits;
- scripts;
- code comments;
- configuration examples;
- architecture decisions;
- troubleshooting guides;
- portfolio descriptions.

Each major document should include the following learning context:

```text
Module
Topic
Subtopic
```

Where appropriate, technical documents also include:

- purpose;
- scope;
- environment;
- prerequisites;
- procedure;
- validation;
- troubleshooting;
- security considerations;
- backup considerations;
- recovery considerations;
- outcome;
- lessons learned.

Documentation should describe what was actually tested and verified.

Unverified assumptions should not be presented as confirmed results.

---

## Knowledge Base

The central Project Phoenix knowledge base is located at:

```text
docs/project-phoenix-knowledge-base.md
```

The knowledge base is continuously extended.

New learning periods add new sections rather than replacing earlier documentation.

This creates a long-term technical record showing:

- learning progression;
- operational decisions;
- troubleshooting history;
- recovery workflows;
- infrastructure improvements;
- engineering habits.

---

## Reference Documentation

Current reference material includes:

```text
docs/ssh-commands-reference.md
```

Reference files are intended to provide concise operational information that can be used during practical administration.

---

## Runbooks

The `runbooks/` directory contains practical operational documentation created during Project Phoenix labs.

Current runbooks include:

```text
backup-and-restore-fundamentals.md
boot-process-and-recovery-fundamentals.md
filesystem-monitoring-and-capacity-management.md
logging-and-logrotate-lab.md
package-management-and-repository-safety.md
process-limits-and-resource-control.md
safe-system-update-phoenix-linux-01.md
scheduled-tasks-and-automation-lab.md
service-user-and-shared-directory-permissions.md
ssh-access-lenovo2.md
ssh-hardening-phoenix-linux-01.md
storage-lvm-and-persistent-mounts.md
system-health-and-failed-service-troubleshooting.md
systemd-service-and-timer-lab.md
```

These documents demonstrate practical experience with:

- Linux administration;
- troubleshooting;
- system recovery;
- secure access;
- service management;
- storage;
- logging;
- automation;
- resource control;
- package maintenance.

---

## Git Workflow

Visual Studio Code is the primary Git interface for Project Phoenix.

GitHub authentication uses a dedicated Ed25519 SSH key.

The workflow is:

```text
Review changes
    ↓
Stage related files
    ↓
Write a focused commit message
    ↓
Commit locally
    ↓
Sync changes with GitHub
    ↓
Verify repository state
```

Commits should be:

- small;
- focused;
- descriptive;
- technically accurate;
- easy to review;
- connected to one meaningful change.

---

## Commit Message Convention

Project Phoenix uses concise commit messages based on the type of change.

Examples:

```text
docs: add boot process and recovery fundamentals lab
docs: add package management and repository safety runbook
docs: update Project Phoenix knowledge base
chore: add repository gitignore
fix: correct persistent mount documentation
refactor: reorganize runbook structure
feat: add system health verification script
```

Common prefixes include:

| Prefix | Purpose |
|---|---|
| `docs:` | Documentation changes |
| `chore:` | Repository maintenance |
| `fix:` | Corrections |
| `feat:` | New functionality |
| `refactor:` | Structural improvement without changing behavior |
| `test:` | Tests and validation |
| `security:` | Security-related changes |

---

## Current `.gitignore` Policy

The repository excludes common operating system, editor, temporary, and secret files.

Examples include:

```text
.DS_Store
Thumbs.db
.vscode/
*.swp
*.tmp
*.log
.env
.env.*
```

Secrets, credentials, private keys, tokens, and environment files must never be committed.

---

## Security Principles

Project Phoenix treats security as a core engineering requirement.

Security practices include:

- SSH key authentication;
- least-privilege access;
- dedicated service users;
- controlled group membership;
- restricted file permissions;
- configuration backups before risky changes;
- console access before boot-critical work;
- trusted package repositories;
- package signature verification;
- no secrets in Git;
- no private keys in repositories;
- review of listening services;
- review of failed units;
- safe testing in dedicated lab VMs.

Security changes should be documented together with their operational and recovery impact.

---

## Backup and Recovery Principles

Every important infrastructure change should consider:

- what could fail;
- how failure would be detected;
- what must be backed up;
- how the previous state can be restored;
- whether console access is available;
- whether a reboot could cause loss of access;
- how recovery will be validated.

Project Phoenix uses:

- Proxmox snapshots for short-term lab rollback;
- file-level configuration backups;
- documented restore procedures;
- persistent logs;
- direct console access;
- direct IP and port access for emergency recovery;
- Git history for documentation and script recovery.

Snapshots are not treated as replacements for independent backups.

---

## Troubleshooting Method

Project Phoenix follows a structured troubleshooting process:

```text
1. Define the symptom.
2. Confirm the current state.
3. Collect logs and evidence.
4. Identify the affected component.
5. Separate historical messages from current failures.
6. Form a testable hypothesis.
7. Perform the smallest safe test.
8. Validate the result.
9. Restore the previous state when necessary.
10. Document the cause, fix, and prevention.
```

Troubleshooting should avoid random changes.

The objective is to understand the system before modifying it.

---

## Validation Standards

A change is not considered complete until it has been validated.

Validation may include:

- checking command exit status;
- reviewing systemd service state;
- checking logs;
- confirming network connectivity;
- confirming listening ports;
- confirming filesystem mounts;
- confirming permissions;
- testing backup restoration;
- checking package integrity;
- confirming Git status;
- verifying the remote GitHub repository.

Successful command execution alone does not always prove that the intended system behavior is correct.

---

## Module Roadmap

### Module 1 — Linux Foundations

Status:

```text
Completed
```

Focus:

- Linux basics;
- filesystem;
- users;
- permissions;
- processes;
- services;
- networking;
- package basics.

### Module 2 — Linux Administration

Status:

```text
Completed
```

Focus:

- secure SSH administration;
- systemd;
- storage;
- LVM;
- logs;
- backups;
- resource control;
- boot recovery;
- package safety.

### Module 3 — Git & Technical Documentation

Status:

```text
In progress
```

Focus:

- professional Git workflow;
- repository organization;
- Markdown;
- commit quality;
- documentation standards;
- portfolio presentation.

### Module 4 — Docker Platform Engineering

Planned focus:

- Docker architecture;
- images;
- containers;
- volumes;
- networks;
- Compose;
- security;
- backup;
- recovery;
- production-style service organization.

### Module 5 — Automation with Bash, Fish, and Ansible

Planned focus:

- shell scripting;
- safe automation;
- idempotency;
- configuration management;
- Ansible inventory;
- playbooks;
- roles;
- secrets management.

### Module 6 — Monitoring with Prometheus, Grafana, and Loki

Planned focus:

- metrics;
- dashboards;
- alerting;
- log aggregation;
- service health;
- infrastructure visibility.

### Module 7 — CI/CD with GitHub Actions

Planned focus:

- workflow automation;
- testing;
- linting;
- documentation validation;
- deployment pipelines;
- secrets and permissions.

### Module 8 — Infrastructure as Code

Planned focus:

- Terraform or OpenTofu;
- providers;
- state;
- modules;
- reproducibility;
- change planning;
- infrastructure lifecycle.

### Module 9 — Cloud Engineering

Planned focus:

- cloud fundamentals;
- identity and access;
- networking;
- compute;
- storage;
- monitoring;
- cost awareness;
- security.

### Module 10 — Kubernetes

Planned focus:

- cluster architecture;
- workloads;
- services;
- configuration;
- storage;
- observability;
- security;
- troubleshooting.

---

## Portfolio Evidence

This repository is intended to demonstrate:

- practical Linux administration;
- infrastructure troubleshooting;
- secure remote administration;
- systemd service management;
- storage and LVM administration;
- backup and recovery planning;
- logging and monitoring fundamentals;
- safe package maintenance;
- professional documentation;
- Git version-control discipline;
- structured technical learning;
- real HomeLab experience.

The portfolio emphasizes verified practical work rather than theoretical claims.

---docs: rewrite main project README

## Professional Development Goal

The long-term objective is to build the practical knowledge, engineering habits, documentation quality, and portfolio required for international roles in:

- Linux infrastructure;
- systems administration;
- infrastructure engineering;
- platform operations;
- DevOps;
- cloud operations;
- technical support engineering.

---

## Project Philosophy

Project Phoenix is not based on collecting commands.

It is based on understanding systems.

The goal is to develop the ability to:

- observe infrastructure;
- identify failures;
- explain system behavior;
- make safe changes;
- automate repeatable work;
- recover from mistakes;
- document decisions;
- communicate clearly with other engineers.

---

## Repository Status

```text
Repository: project-phoenix
Primary branch: main
Documentation language: English
Primary Git interface: Visual Studio Code
Remote platform: GitHub
Authentication: SSH
Current module: Module 3 — Git & Technical Documentation
```

---

## License

A license has not yet been selected for this repository.

Before scripts, templates, or documentation are intended for external reuse, an appropriate open-source license should be reviewed and added.

---

## Author

Project Phoenix is maintained as a structured Linux Infrastructure and DevOps engineering portfolio.

The repository documents practical progress, technical decisions, troubleshooting exercises, recovery workflows, and infrastructure engineering development.