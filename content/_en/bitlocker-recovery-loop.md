---
title: "BitLocker が毎回回復キーを要求する"
errorText: "BitLocker が毎回回復キーを要求する"
tags: ["Windows", "BitLocker", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)  
  BitLocker prompts for the recovery key on every boot, preventing normal access to the encrypted drive.  
- Impact (1 line)  
  Interrupts user work, breaks remote device recovery, and increases operational overhead.  
- Urgency (High/Medium/Low, 1-line reason)  
  High — causes system inaccessibility and key-management exposure risk for enterprise devices.

## Common causes (priority order)
1. TPM and OS state mismatch (firmware update or BIOS/UEFI setting change).  
2. Boot configuration or Secure Boot modifications, or bootloader replacement.  
3. Group Policy / key-management (MBAM/Intune) misconfiguration or sync failure.  
4. File locks or startup access failures (required files inaccessible during boot).  
5. Execution control blocks (AppLocker / SRP / SmartScreen).

## 60-second checks (quickest steps)
- Launch elevated and retry.  
- Check current ACL:
  ```cmd
  icacls "C:\Windows\System32"
  ```
- Verify your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender settings (including CFA):
  ```powershell
  Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
  ```
- Verify TEMP folder permissions:
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  ```
- Snapshot GPO impact:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5-minute remediation (practical flow)
1) Identify affected device and confirm recent firmware/BIOS updates and GPO changes.  
2) Grant temporary ownership/permissions if missing (commands below).  
3) Whitelist blocked executables in Controlled Folder Access or EDR, or remove Zone.Identifier to retry.  
   - PowerShell (unblock file):
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: Check "Unblock" in file Properties.  
4) If file is locked, terminate the holding process or reboot into Safe Mode to perform the fix.  
⚠ WARNING: If you temporarily disable security features (CFA/EDR), limit the window and re-enable immediately after. Rollback: re-enable the settings using the original management tool.

## Long-term fixes
- Enforce change control for TPM firmware and UEFI/BIOS changes.  
- Centralize BitLocker recovery keys in AD/Intune and monitor key synchronization.  
- Maintain Defender/EDR allowlists and exception policies.  
- Avoid executing from shares or ZIPs; copy installers to local (e.g., C:\Temp) before running.

## Detailed steps (Windows 10/11)
1. GUI permission change: target folder → Properties → Security → Advanced → change owner → verify inheritance.  
2. Command examples (run as admin)
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   Rollback example (return owner to SYSTEM):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore excessive permissions to defaults (inheritance/ACL repair):
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: check state and add allowed app
   ```powershell
   Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
   ```
4. If a file is locked (how to find)
   - Win + R → resmon → CPU tab → Associated Handles → search path → stop the process or reboot into Safe Mode to proceed.  
⚠ WARNING: Do not perform broad ownership changes in system directories. Rollback: set owner back to SYSTEM and reset ACLs.

## Common pitfalls (do not do)
- Change ownership broadly under Windows/Program Files/System32.  
- Grant Everyone:Full on shared folders.  
- Leave security features disabled for extended periods.

## Recurrence prevention checklist
- Operate as standard user and elevate only when necessary.  
- Centralize BitLocker key management and back up regularly.  
- Document changes (including firmware/BIOS) and require reviews.  
- Execute installers immediately after download; avoid running from network/archives.

## Related cases
- Steps when Windows Update returns 0x80070005.  
- Steps when Office/Outlook activation returns 0x80070005.  
- Steps when OneDrive/sync client returns 0x80070005.

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult