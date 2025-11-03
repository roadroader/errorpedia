---
title: "Windows のライセンス認証に失敗 0xC004F074"
errorText: "Windows のライセンス認証に失敗 0xC004F074"
tags: ["Windows", "Activation", "KMS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)  
  Activation fails with error 0xC004F074 (KMS server not found or no response).
- Impact (1 line)  
  Windows becomes unactivated, affecting features, updates, and license compliance.
- Urgency (High, reason)  
  High (unactivated systems risk functionality limits and compliance issues; act promptly).

## Common causes (priority order)
1. Inability to reach the KMS server (missing DNS SRV record, network/firewall block)
2. Product key type mismatch (mixing KMS and MAK, incorrect volume license application)
3. Time sync or domain membership issues (clock skew or domain trust failures)
4. Temporary file or installer file lock preventing the activation flow
5. Execution controls blocking the process (AppLocker / SRP / SmartScreen)

## 60-second checks (fastest steps)
- Run an elevated command prompt → retry activation
- Check license status:
  ```cmd
  slmgr /dli
  slmgr /dlv
  slmgr /xpr
  ```
- Test KMS response (replace domain):
  ```cmd
  nslookup -type=SRV _vlmcs._tcp.yourdomain.local
  ```
- Check current ACLs:
  ```cmd
  icacls "C:\\Path\\To\\Folder"
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
  gpresult /h "%USERPROFILE%\\Desktop\\gp.html"
  ```

## 5-minute fix (practical flow)
1) Open network path → verify KMS SRV record and TCP 1688. If reachable, attempt reactivation:
   ```cmd
   slmgr /ato
   ```
2) Confirm key type → apply correct key if needed:
   ```cmd
   slmgr /ipk <Your-Product-Key>
   slmgr /ato
   ```
3) If time sync is the issue, resynchronize:
   ```cmd
   w32tm /resync
   ```
4) If Defender/EDR blocks activation, add an allow or temporary exclusion (restore later):
   ```powershell
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\\Windows\\System32\\slmgr.vbs"
   ```
5) If files are locked, terminate the locking process or reboot into safe mode and retry.

⚠ Note: Disable firewall/EDR only temporarily for troubleshooting. Rollback: immediately re-enable firewall/restore EDR policies after work.

## Permanent mitigations
- Monitor KMS infrastructure (SRV records, response times, port monitoring).
- Standardize license assignment process and distinguish MAK vs KMS usage.
- Enforce NTP/time-sync monitoring and automatic correction.
- Maintain Defender/EDR exception policies for activation components.
- Avoid running installers directly from shares or archives; copy locally (e.g., C:\Temp) before execution.

## Detailed steps (Windows 10/11)
1. GUI checks  
   - Open Settings → Update & Security → Activation and review error details.  
   - In Services (services.msc), confirm "Software Protection" (sppsvc) is running; start it if stopped.
2. Command examples (run elevated)
   ```cmd
   slmgr /dli
   slmgr /dlv
   slmgr /xpr
   slmgr /ipk <Product-Key>
   slmgr /ato
   nslookup -type=SRV _vlmcs._tcp.yourdomain.local
   ```
   ⚠ Note: Stopping sppsvc to test is not recommended (it can disable licensing features). Rollback:
   ```cmd
   sc config sppsvc start= auto
   net start sppsvc
   ```
3. Fix ownership/permissions when installer access fails
   ```cmd
   takeown /F "C:\\Path\\To\\Folder" /R /D Y
   icacls "C:\\Path\\To\\Folder" /grant Administrators:F /T
   ```
   Rollback example (restore to SYSTEM):
   ```powershell
   $p = "C:\\Path\\To\\Folder"
   icacls $p /setowner "NT AUTHORITY\\SYSTEM" /T
   ```
   Restore excessive permissions (inheritance/ACL repair):
   ```cmd
   icacls "C:\\Path\\To\\Folder" /inheritance:e
   icacls "C:\\Path\\To\\Folder" /reset
   ```
4. Defender: checks and allowlisting
   - Status check:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allow:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\\Windows\\System32\\slmgr.vbs"
     ```
5. If a file is locked, identify it:
   - Run "Win + R" → resmon → CPU tab → Associated Handles → search path → stop process or reboot into safe mode.

⚠ Note: Re-enable any temporarily disabled protections immediately after troubleshooting.

## Common mistakes (do not do)
- Broad ownership changes on Windows/Program Files/System32.
- Granting Everyone:Full on shared folders.
- Permanently opening firewall ports instead of applying proper rules.

## Recurrence checklist
- Configure KMS server availability alerts.
- Audit NTP synchronization health.
- Document and review EDR/Defender exception policies.
- Formalize license distribution and update procedures.

## Related cases
- 0xC004F074 after KMS host migration
- Activation timeouts on isolated clients
- Activation failures caused by time/domain mismatch

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult