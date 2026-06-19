# Windows Powershell

## Usage

### Files and folders

#### Remove folder with recurse

```powershell
Remove-Item -path {path\to\folder}
```

#### Make a symbolic link (CMD)

```cmd
mklink /d \{Symlink name} C:\{Path to dir}
```

### Processes and services

#### Find process ID by port

```powershell
netstat -ano | findstr {port}
```

#### Kill process by ID

```powershell
taskkill /F /PID {ID}
```

#### Kill process on specific port

```powershell
Get-NetTCPConnection -LocalPort {port} | Stop-Process -Force
```

Or find and kill in two steps:

```powershell
$pid = (Get-NetTCPConnection -LocalPort {port}).OwningProcess
Stop-Process -Id $pid -Force
```

Using CMD with netstat:

```cmd
netstat -ano | findstr :{port}
taskkill /PID {PID} /F
```
