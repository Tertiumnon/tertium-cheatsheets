# ZSH

## Installation

### Install ZSH (for current user)

- Install ZSH (Ubuntu).

```bash
sudo apt install zsh
```

- Install Oh My ZSH.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

- Set ZSH as default shell (CentOS).

```bash
sudo chsh -s /bin/zsh {username}
```

## ZSH plugins

### Enable SSH agent in ZSH

- Edit .zshrc

```text
plugins=(... ssh-agent)
```

IMPORTANT: put these settings before the line that sources oh-my-zsh:

```text
zstyle :omz:plugins:ssh-agent identities id_rsa id_rsa2 id_github
```

### Enable ZSH plugin for Visual Studio Code

- Edit ~/.zshrc

```bash
plugins=(
  vscode
)
```

- Run

```bash
vsc
```

[zsh-vscode](https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins/vscode)

### Enable ZSH plugin for NVM

Download plugin.

```bash
git clone https://github.com/lukechilds/zsh-nvm ~/.oh-my-zsh/custom/plugins/zsh-nvm
```

Add next lines to your ZSH config file.

```text
plugins+=(zsh-nvm)
```

[zsh-nvm](https://github.com/lukechilds/zsh-nvm)

## ZSH and UTF-8

- Edit ~/.zshrc

```bash
export LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
```
