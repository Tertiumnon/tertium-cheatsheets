# Windows

## Usage

### Check system files through CHDSK

```powershell
chkdsk /r /f
```

### Check system files through SFC

SFC will scan all protected system files, and replace corrupted files with a cached copy that is located in a compressed folder at %WinDir%\System32\dllcache.

```powershell
SFC /scannow
```

### Check system files through DISM

DISM uses Windows Update to provide the files that are required to fix corruptions.

- CheckHealth

```powershell
DISM /online /cleanup-image /CheckHealth
```

- ScanHealth

```powershell
DISM /online /cleanup-image /ScanHealth
```

- RestoreHealth

```powershell
DISM /online /cleanup-image /RestoreHealth
```

- AnalyzeComponentStore

```powershell
DISM /online /cleanup-image /AnalyzeComponentStore
```

- StartComponentCleanup

```powershell
DISM /online /cleanup-image /StartComponentCleanup
```
