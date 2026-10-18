# Common UNIX Commands Cheatsheet

## Basic User Setup

### 1. Create a New User

```bash
sudo useradd -m -d /home/{user} -s /bin/bash {user}
sudo passwd {user}
```

- `-m`: Create home directory.
- `-d`: Specify home directory path.
- `-s`: Set default shell.

### 2. Add User to Sudoers Group

The command to add a user to the sudoers group varies by distribution. The general command is:

```bash
sudo usermod -aG {sudo_group_name} {user}
```

Replace `{sudo_group_name}` with the appropriate group for your distribution (e.g., `wheel` for AlmaLinux/Arch Linux, `sudo` for Ubuntu). Refer to the distribution-specific cheatsheet for the exact command.

### 3. Allow Passwordless Sudo (Optional)

Edit the sudoers file:

```bash
sudo visudo
```

Add the following line at the end of the file:

```text
{user} ALL=(ALL) NOPASSWD: ALL
```

### 4. Switch to New User

```bash
su - {user}
```

---

## User Management

- **List users:**

  ```bash
  less /etc/passwd
  ```

- **Delete user (and their home directory):**

  ```bash
  sudo userdel -r {user}
  ```

---

## Process Management

- **Find process using a port (lsof):**

  ```bash
  sudo lsof -i TCP:{port}
  ```

- **Find process using a port (netstat):**

  ```bash
  # netstat might need to be installed (e.g., package 'net-tools')
  netstat -tulnp | grep ':{port}'
  ```

---

## File and Directory Management

- **View size of a directory:**

  ```bash
  du -sh {path_to_directory}
  ```

- **Find largest files/directories in current path (top 10):**

  ```bash
  du -a . | sort -n -r | head -n 10
  ```

- **Remove files by pattern (use with caution):**

  ```bash
  # Example: remove all .log files in the current directory only (not subdirectories)
  find . -maxdepth 1 -type f -name "*.{extension}" -delete
  ```

- **Copy file with current datetime in its name:**

  ```bash
  cp {file} {file}-$(date +%Y%m%d-%H%M%S)
  ```
