---
title: "Outlook データ ファイルにアクセスできません 0x8004010F"
errorText: "Outlook データ ファイルにアクセスできません 0x8004010F"
tags: ["Outlook", "Profile", "Exchange"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)
  - Error occurs when Outlook cannot access the default PST/OST file or the profile points to an invalid path (0x8004010F).
- Impact (1 line)
  - Send/receive failures, profile sync failure, Outlook error on startup.
- Urgency (High/Medium/Low, 1-line reason)
  - Medium: Email flow stops and requires prompt remediation, but data loss is avoidable with proper steps.

## Common causes (in priority order)
1. Corrupted profile or PST/OST path mismatch (invalid reference).
2. Missing NTFS ownership or access permissions for the file/folder.
3. Corrupted Outlook data file (header/size corruption).
4. File lock (held by another process, antivirus scan, or sync client).
5. Blocking by AppLocker / Software Restriction Policies (SRP) / SmartScreen or similar execution controls.

## 60-second checks (quickest steps)
- Launch as administrator → retry Outlook
- Check current ACLs:
  ```cmd
  icacls "C:\Path\To\Folder"
  ```
- Check your privileges:
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

## 5-minute remediation (practical flow)
1) Identify the target → grant ownership/permissions (GUI or commands).  
2) Register Outlook/data file and related processes as allowed in Controlled Folder Access or EDR; remove MOTW (Zone.Identifier) if present.  
   - PowerShell (unblock):
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: Properties → check "Unblock" and apply.  
3) If file is locked, reboot or use Safe Mode, or safely terminate the holding process.

## Permanent fixes (root cause mitigation)
- Correct ownership and inheritance locally; avoid broad permission changes.
- Standardize Defender/EDR allow-lists for Outlook and related processes.
- Require submission and approval for GPO/EDR exceptions; document exceptions.
- Avoid running installers directly from shares or ZIPs; copy to local (e.g., C:\Temp) before executing.

## Detailed steps (Windows 10/11)
1. GUI: adjust owner, inheritance, and permissions via Properties → Security → Advanced.
2. Command examples (run elevated):
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   Rollback example (restore SYSTEM):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Reset excessive permissions / restore inheritance:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: check state and add allowed app
   - Check:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allowed application:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
4. If file is locked (how to find)
   - Win + R → resmon → CPU tab → search Associated Handles for the path → stop the process or reboot/safe mode.  
⚠ Note: If you temporarily disable antivirus/CFA to repair, re-enable defenses immediately after completion (rollback: re-enable protections).

## Common pitfalls (what not to do)
- Change broad ownership on Windows, Program Files, or System32 roots.
- Grant Everyone:Full on shared folders.

## Recurrence prevention checklist
- Operate as standard user and elevate only when necessary.
- Run installers immediately after download; avoid executing from ZIPs/network.
- Document and review change procedures and exceptions.

## Related cases
- Steps when Windows Update returns 0x80070005
- Steps when Office/Outlook activation returns 0x80070005
- Steps when OneDrive/sync client returns 0x80070005

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult