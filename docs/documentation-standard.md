# Project Phoenix Documentation Standard

## Learning Context

| Field | Value |
|---|---|
| Module | 3 — Git & Technical Documentation |
| Topic | Markdown Documentation Standards |
| Subtopic | Structure, Style, Validation, and Maintenance |
| Status | Active standard |

## Purpose

This document defines the minimum documentation standard for Project Phoenix.

The goal is to keep technical documentation:

- consistent;
- readable;
- technically accurate;
- easy to maintain;
- useful during troubleshooting;
- suitable for a public engineering portfolio.

---

## Required Learning Labels

Every learning document must include:

```text
Module
Topic
Subtopic
```

These labels should appear near the beginning of the document.

Example:

```markdown
## Learning Context

| Field | Value |
|---|---|
| Module | 3 — Git & Technical Documentation |
| Topic | Markdown Documentation Standards |
| Subtopic | Structure, Style, Validation, and Maintenance |
```

---

## Recommended Document Structure

Use the following structure when it fits the topic:

```text
Title
Learning Context
Purpose
Environment
Prerequisites
Procedure
Validation
Troubleshooting
Security Considerations
Backup and Recovery
Outcome
Lessons Learned
```

Not every document requires every section.

Do not add empty sections only to satisfy the template.

---

## Title Standard

Use one clear level-one heading:

```markdown
# Descriptive Document Title
```

Avoid vague titles such as:

```text
Notes
Test
Linux Commands
Lab
```

Prefer:

```text
SSH Hardening on Ubuntu Server
Persistent LVM Mount Configuration
Systemd Service Failure Troubleshooting
```

---

## Heading Hierarchy

Use headings in logical order:

```markdown
# Document Title

## Main Section

### Subsection

#### Detailed Subsection
```

Do not skip heading levels without a clear reason.

Avoid using headings only for visual size.

---

## Writing Style

Use professional English.

Prefer:

- direct sentences;
- active voice;
- specific technical terms;
- verified observations;
- concise explanations;
- clear cause-and-effect relationships.

Avoid:

- unnecessary marketing language;
- unsupported claims;
- excessive repetition;
- vague statements;
- conversational filler;
- unexplained abbreviations.

Example:

```text
The service failed because the configuration file contained an invalid path.
```

Avoid:

```text
The service somehow had a problem and did not work correctly.
```

---

## Command Formatting

Use fenced code blocks for commands:

```bash
systemctl --failed
```

Use one to three commands per practical step when the document is intended as a guided lab.

Explain what the command does before showing it.

Example:

```markdown
Check whether any systemd units are currently failed:

```bash
systemctl --failed
```
```

Do not include shell prompts unless the prompt itself is relevant.

Prefer:

```bash
sudo apt update
```

Instead of:

```text
igor@phoenix-linux-01:~$ sudo apt update
```

---

## Output Formatting

Use `text` code blocks for command output:

```text
0 loaded units listed.
```

Only include the relevant part of long output.

Do not remove lines that are necessary to understand the result.

---

## Configuration Examples

Use the correct language identifier when possible.

Examples:

```ini
[Service]
Restart=on-failure
```

```yaml
services:
  web:
    image: nginx:latest
```

```json
{
  "enabled": true
}
```

```bash
#!/usr/bin/env bash
set -euo pipefail
```

---

## File Paths and Commands

Use inline code formatting for:

- file paths;
- commands mentioned inside sentences;
- package names;
- service names;
- usernames;
- hostnames;
- configuration directives.

Examples:

```text
`/etc/fstab`
`systemctl daemon-reload`
`openssh-server`
`phoenix-linux-01`
```

---

## Tables

Use tables for structured comparisons and summaries.

Example:

| Check | Result |
|---|---|
| Package database | Healthy |
| Failed units | None |
| Reboot required | No |

Do not use large tables when a short paragraph is easier to read.

---

## Lists

Use lists for:

- prerequisites;
- steps;
- checks;
- risks;
- outcomes.

Keep list items parallel in style.

Example:

- verify the current configuration;
- create a backup;
- apply one controlled change;
- validate the result;
- document the outcome.

---

## Procedure Standard

A practical procedure should clearly separate:

1. preparation;
2. change;
3. validation;
4. rollback.

Example:

```text
Create backup
    ↓
Apply one change
    ↓
Validate configuration
    ↓
Test service
    ↓
Restore backup if validation fails
```

Avoid combining unrelated changes in one step.

---

## Validation Standard

Every meaningful change must include validation.

Examples:

```bash
systemctl is-active ssh
```

```bash
findmnt /mnt/phoenix-data
```

```bash
sudo nginx -t
```

```bash
sudo apt-get check
```

A successful command exit does not always prove that the intended behavior is correct.

Validation should test the actual result.

---

## Troubleshooting Standard

Troubleshooting sections should include:

- symptom;
- likely cause;
- diagnostic command;
- interpretation;
- safe corrective action;
- validation after the fix.

Example:

```markdown
### Symptom

The service does not start.

### Diagnostic Check

```bash
journalctl -u example.service -b
```

### Likely Cause

The configured file path does not exist.

### Corrective Action

Correct the path, reload systemd, and restart the service.

### Validation

```bash
systemctl is-active example.service
```
```

---

## Security Considerations

Document security impact when relevant.

Consider:

- permissions;
- ownership;
- authentication;
- exposed ports;
- secrets;
- repository trust;
- privilege escalation;
- service accounts;
- network access;
- log sensitivity.

Never include:

- passwords;
- API keys;
- private SSH keys;
- recovery phrases;
- tokens;
- private certificates;
- sensitive environment files.

Use placeholders:

```text
<username>
<server-ip>
<domain-name>
<token>
```

---

## Backup and Recovery Standard

Before risky changes, document:

- what is being backed up;
- backup location;
- restore command;
- recovery access method;
- validation after restore.

Example:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Possible restore command:

```bash
sudo cp /etc/fstab.backup /etc/fstab
```

Do not present a rollback command without explaining when it is safe to use.

---

## Evidence Standard

Clearly separate:

```text
Observed
Inferred
Expected
Not tested
```

Example:

```text
Observed:
The service started successfully.

Inferred:
The previous failure was caused by the invalid configuration path.

Not tested:
Automatic recovery after a full host reboot.
```

Do not present an inference as a verified fact.

---

## Screenshot Standard

Screenshots should:

- show only the relevant area;
- hide personal or sensitive data;
- use readable resolution;
- include a short caption;
- be stored in `assets/`;
- use descriptive file names.

Example:

```text
assets/vscode-source-control-clean-state.png
```

Avoid names such as:

```text
Screenshot1.png
image-final-new.png
test.png
```

---

## Diagram Standard

Diagrams should:

- use clear labels;
- show direction of traffic or dependency;
- use consistent naming;
- match the documented environment;
- include a short explanation;
- be stored in `diagrams/`.

Example:

```text
diagrams/project-phoenix-lab-network.png
```

---

## File Naming Standard

Use lowercase kebab-case:

```text
boot-process-and-recovery-fundamentals.md
package-management-and-repository-safety.md
ssh-hardening-phoenix-linux-01.md
```

Avoid:

```text
Boot Process.md
new_document.md
FINAL-V2.md
test1.md
```

---

## Directory Placement

Use:

```text
docs/
```

for knowledge-base content, standards, and references.

Use:

```text
runbooks/
```

for operational procedures and troubleshooting workflows.

Use:

```text
lab/
```

for practical exercises and experiment records.

Use:

```text
scripts/
```

for reusable automation.

Use:

```text
assets/
```

for screenshots and supporting media.

Use:

```text
diagrams/
```

for architecture and infrastructure diagrams.

---

## Commit Standard

Documentation changes should use focused commit messages.

Examples:

```text
docs: add Markdown documentation standard
docs: update SSH troubleshooting runbook
docs: correct LVM recovery procedure
```

Do not use vague messages:

```text
update
changes
fixed stuff
new files
```

---

## Review Checklist

Before committing documentation, verify:

- [ ] Module, Topic, and Subtopic are present.
- [ ] The title is clear.
- [ ] The document is written in professional English.
- [ ] Commands are formatted correctly.
- [ ] Output is separated from commands.
- [ ] File paths use inline code formatting.
- [ ] Observed facts are separated from assumptions.
- [ ] Validation steps are included.
- [ ] Security impact is documented where relevant.
- [ ] Backup or rollback is documented where relevant.
- [ ] No secrets or private data are present.
- [ ] File name uses lowercase kebab-case.
- [ ] The document is stored in the correct directory.
- [ ] The commit message describes one focused change.

---

## Maintenance Standard

Documentation should be updated when:

- system behavior changes;
- a command becomes obsolete;
- a configuration path changes;
- a recovery procedure is improved;
- a mistake is discovered;
- a lab produces new verified evidence.

Existing weekly and knowledge-base documents should be extended rather than replaced.

Old information should be corrected clearly instead of silently duplicated.

---

## Outcome

Project Phoenix now has a shared Markdown and technical writing standard.

Future documentation should be:

```text
consistent
verifiable
secure
recoverable
maintainable
portfolio-ready
```