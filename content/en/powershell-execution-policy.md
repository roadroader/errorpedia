---
title: "PowerShell スクリプトが実行ポリシーでブロック"
errorText: "PowerShell スクリプトが実行ポリシーでブロック"
tags: ["PowerShell", "Policy", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger: Attempting to run a PowerShell script returns an “execution policy blocks this script” error (blocked by local/domain policy or Defender).  
- Impact: Script execution stops; installs/configurations/automation fail to complete.  
- Severity: Medium (work is blocked but usually recoverable quickly via privilege or policy changes).

## Common causes (by priority)
1. Local or machine/user-scoped ExecutionPolicy set to Restricted/AllSigned, preventing execution.  
2. Domain GPO enforcing an execution policy (cannot be overridden locally).  
3. File has a Zone.Identifier (downloaded from internet) and requires Unblock.  
4. File locked by another process causing execution/validation to fail.  
5. Execution control by AppLocker / SRP / SmartScreen / EDR blocking the script.

## 60‑second checks (quickest steps)
- Launch PowerShell as administrator → retry.  
- Check current ACL:
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

## 5‑minute resolution (practical flow)
1) Identify the target file and copy to local disk if needed.  
2) Remove Zone.Identifier or run Unblock-File:
```powershell
Unblock-File -Path "C:\Path\to\script.ps1"
```
3) Temporarily relax execution policy at process scope to test (run as admin):
⚠ Note: Do not permanently relax execution policy; process-scope change reverts when session ends — close session to revert.
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\script.ps1
```
4) If Defender/EDR is blocking, register the app/script as allowed (CFA example) or add an exception in the EDR console. Minimize exceptions after work is done.

## Permanent fixes
- Document ExecutionPolicy standards and enforce consistent GPO settings.  
- Sign scripts and define trusted distribution locations.  
- Maintain Defender/EDR allow-list policies and a request workflow.  
- Avoid running scripts directly from shares or ZIPs; always copy to local path (e.g., C:\Temp) before executing.

## Detailed steps (Windows 10/11)
1. GUI checks: In file Properties, check “Unblock” and confirm signature status.  
2. Command examples (run as administrator):
```cmd
takeown /F "C:\Path\To\Folder" /R /D Y
icacls "C:\Path\To\Folder" /grant Administrators:F /T
```
Rollback example (restore SYSTEM owner):
```powershell
$p = "C:\Path\To\Folder"
icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
```
Restore excessive permissions / fix inheritance:
```cmd
icacls "C:\Path\To\Folder" /inheritance:e
icacls "C:\Path\To\Folder" /reset
```
3. Defender: check state and add allow:
```powershell
Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
```
4. If file is locked (how to find):
- Press Win + R → resmon → CPU → Associated Handles → search path → terminate the holding process or reboot / use safe mode.  
⚠ Note: Verify impact before terminating processes; restore services/processes to normal after work.

## Common mistakes (don’t do)
- Broadly changing ownership under Windows or Program Files/System32.  
- Granting Everyone:Full on shared folders to “fix” execution issues.

## Recurrence checklist
- Operate as standard user and elevate only when necessary.  
- Run installers immediately after download to reduce lock/zone issues.  
- Keep change procedures documented and reviewed.

## Related cases
- Steps when Windows Update returns 0x80070005  
- Steps when Office/Outlook activation returns 0x80070005  
- Steps when OneDrive/sync client returns 0x80070005

## References
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls