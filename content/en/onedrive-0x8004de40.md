---
title: "OneDrive にサインインできません 0x8004DE40"
errorText: "OneDrive にサインインできません 0x8004DE40"
tags: ["OneDrive", "Network", "Proxy"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)
  - Error 0x8004DE40 occurs when OneDrive client fails sign-in due to network/credential or local permission inconsistencies.
- Impact (1 line)
  - Sign-in failure stops sync; user files and settings are not synchronized to the cloud.
- Severity (Medium, 1-line reason)
  - Medium. Individual user recovery is quick, but the issue can affect many users in enterprise environments and should be prioritized.

## Common causes (priority order)
1. Network/proxy configuration prevents reaching OneDrive authentication endpoints (DNS, proxy, firewall).
2. Corrupted saved credentials or browser auth cache causing token negotiation failure.
3. Corrupted OneDrive app configuration or cache (may require reinstall).
4. File/folder locked by another process causing auth or startup to fail.
5. AppLocker / Software Restriction Policy (SRP) / SmartScreen / EDR blocking execution or network calls.

## 60-second checks (quickest steps)
- Launch as Administrator → retry OneDrive sign-in
- Check current ACLs:
  ```cmd
  icacls "C:\Users\%USERNAME%\AppData\Local\Microsoft\OneDrive"
  ```
- Check your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender blocks/history and CFA state:
  ```powershell
  Get-MpThreatDetection -ErrorAction SilentlyContinue | Select Timestamp, Action, Resources | Select -First 5
  Get-MpPreference | Select EnableControlledFolderAccess
  ```
- Check TEMP folder permissions:
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  ```
- Snapshot GPO effects:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5-minute resolution (practical flow)
1) Identify target → grant ownership/permissions with least privilege (GUI or command).
2) Register OneDrive/installer in Controlled Folder Access or EDR exceptions temporarily; remove MOTW (Zone.Identifier) if present.
   - PowerShell (unblock):
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: Check "Unblock" in file Properties for downloaded files.
3) If locked, safely stop the holding process or reboot / boot into Safe Mode and retry.

⚠ Warning: Do not fully disable Defender/EDR. Make temporary adjustments only during work and restore settings immediately after (rollback: revert Defender/EDR settings and re-test).

## Permanent fixes (root cause mitigation)
- Correct ownership and inheritance on OneDrive-related files/folders locally; avoid wide-scope changes.
- Configure Defender/EDR allow lists and exception policies to prevent false positives.
- Establish GPO/EDR approval workflow for installers and apps.
- Avoid executing from network shares or inside ZIP files; copy to local path (e.g., C:\Temp) before running.

## Detailed steps (Windows 10/11)
1. GUI changes
   - Properties → Security → Advanced → change Owner → enable inheritance → assign minimum required permissions.
2. Command examples (run as admin)
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   Rollback example (restore to SYSTEM):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore excessive permissions / inheritance:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: inspect and add allowed apps
   - Check state:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allowed app:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\OneDrive\OneDrive.exe"
     ```
4. If files are locked (how to find)
   - Win + R → resmon → CPU tab → Associated Handles → search path → stop the process or reboot / use Safe Mode.

⚠ Warning: Changing ownership broadly can break OS or apps. Rollback: promptly restore original owner and permissions if no benefit.

## Common mistakes (do not)
- Changing ownership broadly under Windows/Program Files/System32.
- Granting Everyone:Full on shared folders.

## Recurrence prevention checklist
- Operate as standard user and elevate only when necessary.
- Execute installers immediately after download to avoid locks.
- Document and review any permission or exception changes.

## Related cases
- Steps for 0x80070005 during Windows Update
- Steps for 0x80070005 when activating Office/Outlook
- Steps for 0x80070005 with OneDrive/sync client

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult