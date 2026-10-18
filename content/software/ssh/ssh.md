# SSH Cheatsheet

## 1. Generate SSH Key Pair

```sh
ssh-keygen -t ed25519 -C "{email}"
```

- `-t ed25519`: Use Ed25519 algorithm (recommended).
- `-C`: Comment (usually your email).

**Default location:**  

- Private key: `~/.ssh/id_ed25519`  
- Public key: `~/.ssh/id_ed25519.pub`

---

## 2. Copy Public Key to Server

```sh
ssh-copy-id {user}@{host}
```

Or manually:

```sh
cat ~/.ssh/id_ed25519.pub | ssh {user}@{host} "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## 3. Connect to Server

```sh
ssh {user}@{host}
```

- `{user}`: Your username on the remote server.
- `{host}`: IP address or hostname.

---

## 4. File Permissions

**Private key:**  

```sh
chmod 600 ~/.ssh/id_ed25519
```

**Public key:**  

```sh
chmod 644 ~/.ssh/id_ed25519.pub
```

**Authorized keys on server:**  

```sh
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

---

## 5. Example `.ssh/config`

Create or edit `~/.ssh/config` for easier SSH usage:

```sshconfig
Host {alias}
    HostName {host}
    User {user}
    IdentityFile ~/.ssh/id_ed25519
    Port 22
    ForwardAgent yes
```

**Usage:**

```sh
ssh {alias}
```

---

## 6. Useful SSH Options

- Specify identity file:

  ```sh
  ssh -i ~/.ssh/id_ed25519 {user}@{host}
  ```

- Run a command remotely:

  ```sh
  ssh {user}@{host} "ls -la"
  ```

- Forward a local port:

  ```sh
  ssh -L 8080:localhost:80 {user}@{host}
  ```

---

## 7. Troubleshooting

- Increase verbosity:

  ```sh
  ssh -v {user}@{host}
  ```

- Check permissions if you get "Permission denied (publickey)".

---

**Tip:** Never share your
