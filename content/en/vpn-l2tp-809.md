---
title: "VPN 接続エラー 809 (L2TP/IPsec)"
errorText: "VPN 接続エラー 809 (L2TP/IPsec)"
tags: ["Windows", "VPN", "Networking"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Occurrence (1 line): Error 809 occurs when L2TP/IPsec fails IPsec negotiation (common causes: NAT, ports, services, certificates).  
- Impact (1 line): Remote access or site-to-site VPN cannot be established, blocking access to business systems.  
- Urgency (High/Medium/Low, 1 line reason): High — directly affects normal operations and requires quick remediation.

## Common Causes (by priority)
1. UDP ports 500/4500 or ESP (protocol 50) are blocked by network/firewall.  
2. Server-side IKE/IPsec services (RasMan / IKEEXT) are stopped or L2TP not enabled.  
3. Pre-shared key (PSK) or certificate mismatch/expiration/untrusted CA; NAT traversal (NAT-T) issues.  
4. File-lock prevents client from loading VPN binaries or temp files (e.g., TEMP permission issues or held handles).  
5. AppLocker / SRP / SmartScreen or similar execution control blocking rasphone.exe or related components.

## 60-Second Checks (minimal)
- Run as administrator → retry  
- Check service status (RasMan, IKEEXT):
  ```powershell
  Get-Service RasMan, IKEEXT
  ```
- Check port reachability (specify VPN server IP):
  ```powershell
  Test-NetConnection -ComputerName vpn.example.com -Port 500
  Test-NetConnection -ComputerName vpn.example.com -Port 4500
  ```
- Check current ACLs:
  ```cmd
  icacls "C:\Path\To\Folder"
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
- Snapshot GPO influence:
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```
- Check Event Viewer (System, Application, RasClient, IKEEXT).

## 5-Minute Fix (operational flow)
1) Open ports/protocols → test connectivity  
   - Allow UDP 500, UDP 4500, and ESP in firewall/router. Enable NAT-T on NAT devices.  
2) Start services → reconnect
   ```powershell
   Start-Service RasMan, IKEEXT
   ```
3) Verify/reconfigure credentials  
   - Match PSK or certificates between server and client.  
4) Unlock client files  
   - Fix temp permissions, stop the holding process, or reboot.  
5) Whitelist VPN executables in Controlled Folder Access/EDR and remove MOTW
   - PowerShell (unblock):
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: Properties → check "Unblock"

## Remediation (long-term)
- Document required ports/protocols in network design and implement monitoring.  
- Automate certificate management to avoid expiration.  
- Maintain allowlists for VPN binaries in EDR/GPO with a formal request flow.  
- Avoid running installers from network shares or archives; copy locally (e.g., C:\Temp) before execution.

## Detailed Steps (Windows 10/11)
1. Use GUI to verify/change services, event logs, certificate store, VPN profile properties.  
2. Command examples (run as admin) — set ownership/permissions if needed:
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
3. Defender: check status and add allowed apps
   - Check:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - Add allowed app:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Windows\System32\rasphone.exe"
     ```
4. If a file is locked (how to find)
   - Win + R → resmon → CPU tab → Associated Handles → search path → end process or reboot / use Safe Mode.  
⚠ Note: If temporarily disabling firewall or EDR for troubleshooting, do so only during the test and immediately restore settings. Rollback: record changes and revert them after validation.

## Common Mistakes (do not)
- Permanently disabling VPN-related services (except for temporary debugging).  
- Granting Everyone:Full broadly on servers or clients.  
- Permanently disabling certificate validation.

## Preventive Checklist
- Monitor server certificate expiry.  
- Ensure NAT-T support and logging on network devices.  
- Document and test change procedures in a lab.  
- Periodically review EDR/GPO allowlists.

## Related Cases
- IPSec negotiation failures producing 13876/13894 logs.  
- Concurrent authentication errors (e.g., 691) when L2TP also reports issues.  
- Instability caused by NAT packet rewriting.

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult