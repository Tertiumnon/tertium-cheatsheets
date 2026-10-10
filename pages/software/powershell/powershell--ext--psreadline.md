# PSReadLine

Line editing, history and prediction module for PowerShell. Ships with PowerShell 5.1 and PowerShell 7.

## Check version

```powershell
Get-Module PSReadLine -ListAvailable | Select-Object Name, Version
```

## Update

```powershell
Install-Module PSReadLine -Scope CurrentUser -Force
```

## Show current options

```powershell
Get-PSReadLineOption
```

## Predictions from history

Show inline suggestions from command history:

```powershell
Set-PSReadLineOption -PredictionSource History
```

Accept the suggestion with `→` (right arrow) or `End`.

## Edit mode

Use Emacs-style key bindings:

```powershell
Set-PSReadLineOption -EditMode Emacs
```

Back to Windows-style bindings:

```powershell
Set-PSReadLineOption -EditMode Windows
```

## Key bindings

List all key handlers:

```powershell
Get-PSReadLineKeyHandler
```

Show completion menu on `Tab`:

```powershell
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
```

Search history by the typed prefix with `Up` / `Down`:

```powershell
Set-PSReadLineKeyHandler -Key UpArrow -Function HistorySearchBackward
Set-PSReadLineKeyHandler -Key DownArrow -Function HistorySearchForward
```

## Persist settings

Put the lines above into the profile (`notepad $PROFILE`) so they load on every session.
