---
layout: post
title: "Fixing wsl-vpnkit Auto-start Dying After Login"
category: CS
date: 2026-05-27
tags: [wsl2, vpn, autostart]
---

## Problem
 
I set up wsl-vpnkit auto-start with Task Scheduler in my [previous post](/cs/2026/04/13/wsl2-vpn-internet-fix/), but it turned out the task triggers fine at login and then dies about a minute later.
 
```
PS> Get-ScheduledTask -TaskName "wsl-vpnkit" | Get-ScheduledTaskInfo
 
LastRunTime    : 2026/05/27 09:04:46
LastTaskResult : 1
```
 
The return code is `2147942401` (`0x80070001`) and the WSL distro ends up `Stopped`, so there's no internet inside WSL until I run wsl-vpnkit manually. Kinda defeats the whole point of auto-start.
 
## Why it dies
 
Task Scheduler runs `wsl.exe` in a non-interactive session with no TTY attached, but wsl-vpnkit is a foreground daemon that needs one. After about a minute the host `wsl.exe` exits, vpnkit dies with it, and the distro shuts down. This is a [known issue](https://github.com/microsoft/WSL/issues/10732).
 
I tried `nohup` to detach from the TTY but the process exited immediately, and wrapping the command in `wt.exe` via Task Scheduler didn't work either because GUI apps don't render in Task Scheduler's non-interactive session.
 
## Fix
 
The Startup folder runs things in the actual user desktop session, same as if you launched it yourself, so just use that instead.
 
First delete the old task:
 
```powershell
Unregister-ScheduledTask -TaskName "wsl-vpnkit" -Confirm:$false
```
 
Then create a shortcut in Startup that opens Windows Terminal and runs wsl-vpnkit:
 
```powershell
$WshShell = New-Object -ComObject WScript.Shell
$StartupPath = [Environment]::GetFolderPath('Startup')
$Shortcut = $WshShell.CreateShortcut("$StartupPath\wsl-vpnkit.lnk")
$Shortcut.TargetPath = "wt.exe"
$Shortcut.Arguments = '--window 0 new-tab --title "wsl-vpnkit" wsl.exe -d wsl-vpnkit --cd /app wsl-vpnkit'
$Shortcut.Save()
```
 
While you're at it, kill the WSL idle timeout so the distro doesn't auto-shutdown when nothing's actively running:
 
```
# %USERPROFILE%\.wslconfig
[wsl2]
networkingMode=nat
vmIdleTimeout=-1
```
 
Reboot and you'll see the terminal pop up at login with vpnkit running.
 
## Trade-off
 
The terminal window stays visible the whole time, which is actually fine since you can tell at a glance whether vpnkit is alive and the logs are right there if something breaks. Just minimize it and forget about it.
