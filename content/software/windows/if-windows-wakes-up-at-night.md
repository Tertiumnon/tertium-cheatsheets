# If your Windows wakes up at night

## Solution 1

It's really annoying when you wake up from the sound of turning on of your PC. I tried to set some options in Windows but with no results.

My last hope was an Internet and I found this solution:

Try to understand why is so happening:

```powershell
powercfg -waketimers
```

You'll see something like that:

```powershell
Timer set by [SERVICE] \Device\HarddiskVolume2\Windows\System32\svchost.exe (SystemEventsBroker) expires at 7:26:28 PM on 11/29/2019.
Reason: Windows will execute 'NT TASK\Microsoft\Windows\UpdateOrchestrator\Backup Scan' scheduled task that requested waking the computer.
Timer set by [PROCESS] \Device\HarddiskVolume2\Windows\SystemApps\Microsoft.Windows.StartMenuExperienceHost_cw5n1h2txyewy\StartMenuExperienceHost.exe expires at 7:26:28 PM on 11/29/2019.
```

In my case it was "\UpdateOrchestrator\Backup Scan". And my next steps were:

* Download [PSTools](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec).
* Extract files.
* Use a command-line in extracted directory to disable this disgusting service:

```powershell
.\PsExec.exe -s schtasks /change /tn "\Microsoft\Windows\UpdateOrchestrator\Backup Scan" /DISABLE
```

## Solution 2

In the moment after your computer was awaked run:

```powershell
powercfg /lastwake
```

You'll see a name of device.

Go to Device Manager: Win + R, then print and press "OK":

```text
devmgmt.msc
```

Find the device and in it's properties click the "Power Management" and untick "Allow this device to wake up the computer" and click OK.
