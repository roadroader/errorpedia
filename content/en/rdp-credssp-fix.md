---
title: "RDP 接続エラー: CredSSP 暗号化オラクルを修復"
errorText: "RDP 接続エラー: CredSSP 暗号化オラクルを修復"
tags: ["Windows", "RDP", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)
  - RDP fails with "Encryption Oracle Remediation (CredSSP)" error due to mismatched CredSSP patches/settings between client and server.
- Impact (1 line)
  - Remote Desktop connections fail, interrupting administration and application access.
- Severity (High, 1-line reason)
  - High (lack of remote access delays recovery and security remediation).

## Common causes (priority order)
1. Client/server Windows updates are not aligned (CredSSP fix applied only to one side).
2. Local/domain GPO enforces strict "Encryption Oracle Remediation" and blocks connection.
3. Certificate or credential issues (NLA or credential cache corruption).
4. File/process lock prevents updating RDP-related binaries (installer failures).
5. AppLocker / SRP / SmartScreen or similar execution control blocking RDP components.

## 60-second check (fastest steps)
- Run as administrator → retry
- Check current ACL:
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
- Verify TEMP permissions:
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  ```
- Snapshot GPO impact:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```
- Check CredSSP setting:
  ```cmd
  reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle
  ```

## 5-minute fix (operational flow)
1) First and best: apply latest Windows updates to both client and target server and reboot both machines — this resolves the mismatch.
2) Temporary workaround (if patching is impossible immediately):
   - Loosen local policy temporarily (security risk):
     ```cmd
     reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 2 /f
     gpupdate /force
     ```
   - GUI: gpedit.msc → Computer Configuration → Administrative Templates → System → Credentials Delegation → Encryption Oracle Remediation → set to "Vulnerable".
   ⚠ Warning: This reduces security. Revert after patching (see rollback).
3) Restart RDP service / reboot target and retry:
   ```cmd
   net stop TermService & net start TermService
   ```
4) If files are locked, stop the holding process or reboot into safe mode and apply updates.

Rollback (one-line):
- After patching, restore setting:
  ```cmd
  reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 0 /f
  gpupdate /force
  ```

## Long-term mitigations (permanent)
- Deploy Windows updates in coordinated manner (clients and servers together).
- Enforce change management for GPO templates and test before rollout.
- Maintain an up-to-date jump server for remote management and avoid direct dependence on legacy hosts.
- Copy installers/tools locally (e.g., C:\Temp) before execution instead of running from network shares.

## Detailed steps (Windows 10/11)
1. GUI verification
   - Check gpedit.msc / Group Policy Management for Encryption Oracle Remediation. In domain environments, request GPO change from administrators.
2. Command examples (run as administrator)
   ```cmd
   reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle
   reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 2 /f
   gpupdate /force
   ```
   Rollback example (return safely):
   ```cmd
   reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 0 /f
   gpupdate /force
   ```
3. Confirm updates and retry
   - Ensure relevant KBs are installed on both ends and reboot.
4. If file is locked (finding)
   - Use Resource Monitor (resmon) → CPU → Associated Handles to find the handle, stop the process or reboot.

⚠ Warning: Temporarily setting policy to "Vulnerable" is a last resort. Revert after applying updates.

## Common mistakes (do not do)
- Permanently operating with AllowEncryptionOracle set to 2 (Vulnerable).
- Updating only clients and leaving servers unpatched.

## Recurrence prevention checklist
- Define synchronized update policy for critical servers and client groups.
- Require ticketing/review for GPO changes.
- Test CredSSP updates in a lab before production rollout.

## Related cases
- Procedures when Windows Update shows 0x80070005
- Procedures when Office/Outlook activation shows 0x80070005
- Procedures when OneDrive/sync client shows 0x80070005

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult