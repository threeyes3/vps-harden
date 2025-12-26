# SSH Hardening Guide (English)

This document is the single, complete guide for using this project.

---

## Before You Start

- Always keep your current SSH session open
- Open the new SSH port in your cloud security group first
- Interactive mode is strongly recommended for first-time use

---

## Recommended Usage

Run the script interactively:

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo bash
```

The script will guide you through key deployment and safety checks.

---

## Fast Mode

Fast mode is intended for automation and advanced users only.

```bash
curl -fsSL https://raw.githubusercontent.com/threeyes3/vps-ssh-harden/main/harden-ssh.sh | sudo \
NEW_PORT=40022 GITHUB_KEYS_USER=threeyes3 bash
```

---

## Recovery

Restore the SSH configuration backup if needed and restart sshd.
