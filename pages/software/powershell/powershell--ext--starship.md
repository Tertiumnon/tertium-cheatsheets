# Starship

Cross-shell prompt written in Rust. Alternative to Oh My Posh.

## Install

```powershell
winget install Starship.Starship
```

## Enable in PowerShell profile

Open the profile:

```powershell
notepad $PROFILE
```

Add the init line:

```powershell
Invoke-Expression (&starship init powershell)
```

## Configure

Config file is read from `$env:STARSHIP_CONFIG` if set, otherwise `~/.config/starship.toml`.

Print the full default config:

```powershell
starship print-config
```

Write a built-in preset to the config file:

```powershell
starship preset nerd-font-symbols -o ~/.config/starship.toml
```

## Debug

Show which modules render and why:

```powershell
starship explain
```
