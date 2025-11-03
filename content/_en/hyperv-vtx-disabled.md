---
title: "Hyper-V を有効化できない（仮想化サポート無効）"
errorText: "Hyper-V を有効化できない（仮想化サポート無効）"
tags: ["Windows", "Hyper-V", "BIOS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Condition (1 line)  
  Enabling Hyper-V fails with "virtualization support disabled" or firmware reports virtualization is disabled.  
- Impact (1 line)  
  Cannot create/start VMs, use WSL2, or run Docker Desktop.  
- Urgency (High/Medium/Low, 1-line reason)  
  High — development, testing, and container workflows stop, requiring immediate action.

## Common causes (by priority)
1. Virtualization in BIOS/UEFI (Intel VT-x / AMD-V) is disabled  
2. OS edition or security features (Windows Home / Core Isolation / Memory Integrity / Device Guard) blocking Hyper-V  
3. Hyper-V feature missing or corrupted in Windows Features  
4. Files or drivers locked / installer lacks execution permission  
5. Blocking by AppLocker / Software Restriction Policies / SmartScreen

## 60-second check (shortest steps)
- Run as Administrator → retry  
- Check ACLs:
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
- Snapshot GPO effects:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5-minute fix (practical flow)
1) Check BIOS/UEFI: reboot → firmware settings → enable Intel VT-x / VT-d / AMD-V / SVM → save and restart.  
2) Verify OS edition: Windows Home does not include Hyper-V (option: upgrade to Pro or use supported alternatives).  
3) Enable Windows feature (admin command):
   ```cmd
   dism /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
   ```
4) Remove security blocks: temporarily disable Core Isolation / Memory Integrity and retry (reboot required).  
   ⚠ Note: Disable Memory Integrity only for diagnosis. Re-enable after work (rollback: re-enable in same settings UI).  
5) If installer/script is blocked, register allowlist or unblock file; try install in Safe Mode or after reboot.

## Permanent measures
- Standardize firmware settings to keep virtualization enabled (manage BIOS configuration)  
- Maintain Defender/EDR allowlists for Hyper-V and related installers  
- Formalize GPO/MDM request flows for virtualization-related policies (Device Guard, Credential Guard)  
- Avoid running installers directly from shares or ZIPs; copy to local (e.g., C:\Temp) before execution

## Detailed steps (Windows 10/11)
1. Enable virtualization in BIOS/UEFI per vendor instructions.  
2. Permission repair examples (run as admin):
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   Rollback example (restore SYSTEM owner):
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   Restore defaults / inheritance:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: check and add allowed apps  
   - Check state:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allow:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
4. If a file is locked (how to find):  
   - Win + R → resmon → CPU → Associated Handles → search path → stop process or reboot / use Safe Mode.  
   ⚠ Note: Killing processes can have significant impact. Verify effects before stopping; restart services if needed (rollback: restart stopped services).

## Common pitfalls (do not)
- Mass changing ownership under Windows/Program Files/System32  
- Granting Everyone:Full on shared folders

## Prevention checklist
- Operate as standard user and elevate only when needed  
- Execute installers locally immediately after download  
- Track firmware changes with configuration management

## Related cases
- Steps when Windows Update returns 0x80070005  
- Steps when Office/Outlook activation returns 0x80070005  
- Steps when OneDrive / sync client returns 0x80070005

## References (primary sources)
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult