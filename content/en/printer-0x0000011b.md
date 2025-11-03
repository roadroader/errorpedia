---
title: "プリンターに接続できません 0x0000011b"
errorText: "プリンターに接続できません 0x0000011b"
tags: ["Windows", "Printer", "Network"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)  
  Windows returns error code "0x0000011b" when sending print jobs to a network/shared printer.  
- Impact (1 line)  
  Printer connection fails, printing stops, shared-printer workflows are interrupted.  
- Severity (Medium — authentication/permission mismatches can block business-critical printing)

## Common causes (by priority)
1. SMB authentication/signing mismatch between client and print server (often after Windows updates).  
2. Print Spooler service permission or process failure (service not responding or access denied).  
3. Network path or name resolution issues (DNS or hostname resolution failure).  
4. File/port locked by another process (spool files or temp files held open).  
5. Blocking by AppLocker / SRP / SmartScreen / other execution controls

## 60‑second checks (quick steps)
- Run an elevated shell → retry connection.  
- Check and restart Print Spooler:
  ```cmd
  sc query spooler
  sc stop spooler
  sc start spooler
  ```
- Inspect current ACL:
  ```cmd
  icacls "C:\Windows\System32\spool\PRINTERS"
  ```
- Verify your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender history and Controlled Folder Access state:
  ```powershell
  Get-MpThreatDetection -ErrorAction SilentlyContinue | Select Timestamp, Action, Resources | Select -First 5
  Get-MpPreference | Select EnableControlledFolderAccess
  ```
- Check TEMP/spool folder permissions and locks:
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  handle -a PRINTERS  (use Sysinternals Handle if available)
  ```
- Snapshot GPO influence:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5‑minute resolution (operational flow)
1) Identify the target → restart the Print Spooler and review logs (Event Viewer: System/Application).  
2) If ownership/ACLs are the cause, temporarily take ownership and grant permissions (run as admin).  
   - Commands (admin):
     ```cmd
     takeown /F "C:\Windows\System32\spool\PRINTERS" /R /D Y
     icacls "C:\Windows\System32\spool\PRINTERS" /grant Administrators:F /T
     ```
   ⚠ Warning: Changing ownership affects system behavior—restore ownership after work (see rollback in Detailed Steps).  
3) Check Defender/EDR and Controlled Folder Access for blocks; add allow entries for spooler/driver files.  
   - Unblock file (PowerShell):
     ```powershell
     Unblock-File -Path "C:\Path\to\printer-driver-installer.exe"
     ```
   - GUI: Check driver/installer Properties → Unblock checkbox.  
4) If a file lock causes the problem, terminate the holding process, restart the service, or reboot/enter Safe Mode to clear locks.

## Permanent mitigations
- Align SMB/authentication policies between print servers and clients; include printing tests in update rollout.  
- Monitor Print Spooler permissions and prevent broad ownership changes.  
- Maintain EDR/Defender allowlists for printer drivers and spool folders.  
- Avoid running installers from network shares; copy installers locally (e.g., C:\Temp) before execution.

## Detailed steps (Windows 10/11)
1. GUI: check target folder owner → enable inheritance → assign minimal required rights to Administrators.  
2. Command examples (run as Administrator)
   ```cmd
   takeown /F "C:\Windows\System32\spool\PRINTERS" /R /D Y
   icacls "C:\Windows\System32\spool\PRINTERS" /grant Administrators:F /T
   ```
   Rollback example (restore to SYSTEM):
   ```powershell
   $p = "C:\Windows\System32\spool\PRINTERS"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore excessive permissions/inheritance:
   ```cmd
   icacls "C:\Windows\System32\spool\PRINTERS" /inheritance:e
   icacls "C:\Windows\System32\spool\PRINTERS" /reset
   ```
3. Defender: check and add allowed applications  
   - Status check:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allow:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Windows\System32\spool\drivers\x64\3\printerdriver.dll"
     ```
4. If files are locked (finding method)  
   - Run Resource Monitor (Win + R → resmon) → CPU tab → Associated Handles → search path → end process or reboot/enter Safe Mode.  
⚠ Warning: Temporarily disabling Defender/EDR or changing ownership should be performed only during work and reverted immediately after completion.

## Common mistakes (do not do)
- Change ownership broadly on System or Program Files folders.  
- Assign Everyone:Full on shared printer folders.

## Recurrence checklist
- Operate with standard user rights and elevate only when needed.  
- Run installers locally after download to avoid network locks.  
- Document and review change procedures.

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