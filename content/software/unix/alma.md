# AlmaLinux Server Setup Cheatsheet

This cheatsheet focuses on AlmaLinux-specific setup steps. For common UNIX/Linux commands related to user creation (including `useradd`, `passwd`), passwordless sudo for a specific user (`visudo`), switching users (`su -`), general user management (`less /etc/passwd`, `userdel`), process management (`lsof`), and file/directory operations (`du`, `find`, `cp`), please refer to `unix-setup.md`.

## AlmaLinux Specific Setup

### 1. Add User to Sudoers (wheel group)

After creating a user (see `unix-setup.md`), add them to the `wheel` group to grant sudo privileges:

```bash
sudo usermod -aG wheel {user}
```

### 2. Allow Passwordless Sudo for a specific user (Optional)

The method for allowing a specific user passwordless sudo by editing the sudoers file with `sudo visudo` is covered in `unix-setup.md`.

---

## System Upgrade

Update all packages on the system:

```bash
sudo dnf update
```
