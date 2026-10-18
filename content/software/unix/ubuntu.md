# Ubuntu Server Setup Cheatsheet

This cheatsheet focuses on Ubuntu-specific setup steps. For common UNIX/Linux commands related to user creation (including `useradd`, `passwd`), passwordless sudo for a specific user (`visudo`), switching users (`su -`), general user management, process management, and file/directory operations, please refer to `unix-setup.md`.

## Ubuntu Specific Setup

### 1. Add User to Sudoers Group

After creating a user (see `unix-setup.md`), add them to the `sudo` group:

```bash
sudo usermod -aG sudo {user}
```

### 2. System Upgrade

Update installed packages:

```bash
sudo apt update && sudo apt full-upgrade
```

For a distribution version upgrade:

```bash
sudo do-release-upgrade
```
