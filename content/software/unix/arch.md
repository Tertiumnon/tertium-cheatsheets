# Arch Linux Server Setup Cheatsheet

This cheatsheet focuses on Arch Linux-specific setup steps. For common UNIX/Linux commands related to user creation, user management, process management, and file/directory operations, please refer to `unix-setup.md`.

## Arch Linux Specific Setup

### 1. Add User to Sudoers (wheel group)

After creating a user (see `unix-setup.md`), add them to the `wheel` group to grant sudo privileges:

```bash
sudo usermod -aG wheel {user}
```

### 2. Configure Passwordless Sudo for `wheel` group (Optional)

To allow all users in the `wheel` group to run commands with `sudo` without a password, edit the sudoers file:

```bash
sudo visudo
```

Uncomment or add the following line:

```text
%wheel ALL=(ALL) NOPASSWD: ALL
```

**Note:** For allowing a specific user passwordless sudo, refer to `unix-setup.md`.

---

## System Upgrade

Update all packages on the system:

```bash
sudo pacman -Syu
```
