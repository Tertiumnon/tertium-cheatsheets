# Oh My Posh

Prompt theme engine for PowerShell.

## Install

```powershell
winget install JanDeDobbeleer.OhMyPosh -s winget
```

## Enable in PowerShell profile

Open the profile:

```powershell
notepad $PROFILE
```

Add the init line:

```powershell
oh-my-posh init pwsh | Invoke-Expression
```

## Use a theme

List built-in themes:

```powershell
Get-PoshThemes
```

Init with a specific theme:

```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\{theme}.omp.json" | Invoke-Expression
```
