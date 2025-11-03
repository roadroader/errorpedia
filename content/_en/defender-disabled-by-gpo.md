---
title: "Microsoft Defender がグループポリシーにより無効化"
errorText: "Microsoft Defender がグループポリシーにより無効化"
tags: ["Windows", "Defender", "Policy"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)  
  Occurs when Microsoft Defender Antivirus is disabled by Group Policy (GPO) or managed policy.
- Impact (1 line)  
  Endpoint real-time protection is stopped, increasing malware/ransomware risk.
- Severity (High/Medium/Low, 1-line reason)  
  High — an unprotected endpoint becomes an immediate attack surface and requires prompt action.

## Common causes (priority order)
1. Organization GPO sets "Disable Windows Defender" policy.  
2. MDM (Intune) or EDR management policy distributed a disable setting.  
3. Tamper Protection prevents local recovery actions.  
4. File/registry permission inconsistencies or locks prevent services from starting (include file lock cause).  
5. AppLocker / Software Restriction Policies (SRP) / SmartScreen or other execution controls blocking components.

## 60-second checks (quickest steps)
- Run elevated and retry.  
- Check current ACL:
  ```cmd
  icacls "C:\Path\To\Folder"
  ```
- Verify your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender detections and CFA status:
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

## 5-minute fix (practical flow)
1) Identify affected endpoints → use gpresult to find which GPOs apply.  
2) If GPO is the cause, request change from policy administrators and confirm organizational intent; adjust or disable the policy as needed.  
3) If local recovery is permitted, inspect and edit registry/local policy (requires admin).  
   ⚠ Warning: Edit the registry or policies only with operator approval. Restore original values after completion.  
   Example (check disable flag):
   ```cmd
   reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware
   ```
   Example (short-term local enable):
   ```cmd
   reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /t REG_DWORD /d 0 /f
   ```
   Rollback: restore previous value or have the policy re-applied from management.  
4) If Tamper Protection or EDR is blocking, change settings in the management console.  
5) Check Defender settings and add application allowances (Controlled Folder Access / exceptions):
   ```powershell
   Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
   ```
6) If files are locked, reboot into Safe Mode or terminate the holding process safely, then retry.

## Remediations (long-term)
- Enforce formal change request flow for GPO/MDM/EDR changes and retain change history.  
- Clarify roles between Tamper Protection and EDR policies.  
- Define criteria for exception registrations and maintain least-privilege operations.  
- Require installers/scripts to be copied locally (e.g., C:\Temp) before execution.

## Detailed steps (Windows 10/11)
1. GUI checks  
   - Review "Windows Defender Antivirus" in gpedit.msc or the relevant GPO.  
   - Verify Tamper Protection in Security Center and whether admins can change it.  
2. Command examples (run as administrator)
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   Rollback example (restore to SYSTEM):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore excessive permissions / repair inheritance:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: state check and allow listing
   ```powershell
   Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
   ```
4. If a file is locked (finding method)
   - Press Win + R → resmon → CPU tab → Associated Handles, search path → stop the process or reboot into Safe Mode to work.  
⚠ Warning: Temporarily disabling Defender is allowed only for the duration of work; always re-enable. Rollback by restoring the original policy value.

## Common pitfalls (do not)
- Broad ownership changes on Windows/Program Files/System32 roots.  
- Granting Everyone:Full on shared folders.  
- Repeated attempts to disable Tamper Protection without proper admin rights.

## Recurrence prevention checklist
- Operate users as standard accounts and escalate only when necessary.  
- Document installation/change procedures and enforce approval reviews.  
- Execute installers immediately after download to avoid lock issues.  
- Periodically audit GPO application with gpresult.

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