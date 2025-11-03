---
title: "Microsoft Store アプリのインストールに失敗 0x80073CF9"
errorText: "Microsoft Store アプリのインストールに失敗 0x80073CF9"
tags: ["Windows", "Store", "App"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Repro condition (1 line)  
  Error 0x80073CF9 occurs and the Microsoft Store / UWP package install aborts.  
- Impact (1 line)  
  Prevents app installation and may block updates or repairs.  
- Urgency (Medium, 1-line reason)  
  Blocks installation tasks and affects productivity but does not immediately stop the entire system.

## Common causes (priority order)
1. Insufficient permissions on package destination or TEMP folder (incorrect ownership/ACL)  
2. Blocking by Microsoft Defender or EDR (Controlled Folder Access or quarantines)  
3. Incomplete download or Zone.Identifier (MOTW) preventing execution  
4. File/handle locked by another process (installer temporary files in use)  
5. Blocking by execution controls such as AppLocker / SRP / SmartScreen

## 60-second checks (quick steps)
- Launch as administrator → retry  
- Check current ACLs:
  ```cmd
  icacls "C:\Path\To\Folder"
  ```
- Verify your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender block/history and CFA state:
  ```powershell
  Get-MpThreatDetection -ErrorAction SilentlyContinue | Select Timestamp, Action, Resources | Select -First 5
  Get-MpPreference | Select EnableControlledFolderAccess
  ```
- Check TEMP folder permissions:
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  ```
- Snapshot GPO impact:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5-minute resolution (practical flow)
1) Identify target → assign ownership/permissions (GUI or command, restrict scope)  
2) Register allowed apps in Controlled Folder Access / EDR and remove MOTW (Zone.Identifier)  
   - PowerShell (unblock):
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: Check "Unblock" on file Properties  
   - Defender allow example:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
3) If locked, reboot / use Safe Mode / safely terminate the holding process (find via resmon)

## Long-term fixes (preventive)
- Correct ownership and inheritance (limit scope)  
- Maintain allowlists and exception policies for Defender/EDR  
- Implement request/approval flows for GPO/EDR-controlled environments  
- Avoid running directly from shares or ZIPs—copy to local (e.g., C:\Temp) before executing

## Detailed procedures (Windows 10/11)
1. GUI adjustments: Properties → Security → Advanced → change owner / verify inheritance.  
2. Command examples (run as administrator)
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   ⚠ Warning: Changing ownership and ACLs broadly (Rollback: restore owner to SYSTEM).  
   Rollback example (restore to SYSTEM):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore excessive permissions / repair inheritance and ACLs:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: settings and allowed apps  
   - Check state:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allow:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
4. If a file is locked (how to find)  
   - Press Win + R → resmon → CPU tab → search Associated Handles for the path → stop the process or reboot / use Safe Mode.  
⚠ Warning: Temporarily disabling Defender/EDR is allowed only for the task. Re-enable after completion (Rollback: restore Defender settings).

## Common pitfalls (do not)
- Change ownership broadly under Windows\Program Files or System32  
- Grant Everyone:Full on shared folders

## Recurrence checklist
- Operate as standard user and elevate only when needed  
- Run installer immediately after download to avoid locks  
- Document and review change procedures

## Related cases
- Steps when 0x80070005 appears during Windows Update  
- Steps when 0x80070005 appears during Office/Outlook activation  
- Steps when 0x80070005 appears with OneDrive/sync clients

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult