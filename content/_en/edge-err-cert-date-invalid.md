---
title: "この接続ではプライバシーが保護されません（証明書の有効期限切れ）"
errorText: "この接続ではプライバシーが保護されません（証明書の有効期限切れ）"
tags: ["Edge", "Certificate", "HTTPS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "en"
---

## Summary
- Trigger (1 line)  
  Browser blocks connection after detecting the server TLS certificate is expired.
- Impact (1 line)  
  HTTPS access to the site is refused; confidential data cannot be transmitted.
- Urgency (High, 1-line reason)  
  High: may indicate misconfigured certificate or MITM risk and can cause service outage—respond immediately.

## Common causes (priority order)
1. Server certificate expired (operator failed to renew).
2. Client system clock skewed significantly (validation fails).
3. Missing/expired intermediate or root CA in the chain.
4. File/lock on local certificate cache or store prevents update.
5. Blocking by AppLocker / SRP / SmartScreen / execution control preventing validation tools or agents.

## 60‑second checks (fastest steps)
- Launch browser as Administrator → retry the URL and view certificate details (check validity period).
- Verify and sync system time:
  ```cmd
  w32tm /query /status
  ```
- Export the certificate from the browser and check dates.
- Quick verify exported certificate:
  ```cmd
  certutil -verify "C:\Path\to\exported-cert.cer"
  ```
- Check current ACLs:
  ```cmd
  icacls "C:\Windows\System32\CertLog"
  ```
- Check your privileges:
  ```cmd
  whoami /groups & whoami /priv
  ```
- Check Defender blocks/history and CFA status:
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

## 5‑minute remediation (operational flow)
1) Client checks: sync time, inspect certificate details (issue/expiry/subject/thumbprint).  
2) Contact server operator: request certificate renewal (reissue and deploy) if expired.  
3) Temporary bypass (only if absolutely necessary):
   - Avoid browser exception; if business-critical, use browser’s proceed option temporarily.  
   ⚠ 注意: Adding a browser exception is dangerous. Remove the exception immediately after task completion.
   - Temporary diagnostic (retrieve cert chain):
     ```cmd
     openssl s_client -connect example.com:443 -servername example.com -showcerts
     ```
4) If intermediate/root missing, add intermediate on server or install proper root on clients (admin rights required).  
   ⚠ 注意: Installing unknown/self-signed CA to trust store is dangerous. Revert after use (rollback steps below).
5) Verify network path for proxy/TLS inspection and coordinate with network/proxy team.

## Permanent fixes (long term)
- Automate certificate renewal (ACME/scheduled tasks).
- Monitor certificate expiry and integrate alerts into operations.
- Enforce NTP time sync across endpoints.
- Deploy full certificate chain (including intermediates) on servers.
- Document update procedures and assign responsible personnel.

## Detailed steps (Windows 10/11)
1. In the browser, view certificate → check Certification Path. If intermediate not shown, fix server config to include it.  
2. Temporarily install a certificate to local store (admin):
   ```cmd
   certutil -addstore "Root" "C:\Path\to\root-ca.cer"
   ```
   Rollback example (remove from Root):
   ```cmd
   certutil -delstore "Root" "THUMBPRINT_OF_CERT"
   ```
3. Verify certificate:
   ```cmd
   certutil -verify "C:\Path\to\exported-cert.cer"
   ```
4. If the store is locked and cannot be updated:
   - Reboot to release holding process or perform operations in Safe Mode.  
   ⚠ 注意: Modify certificate stores only with administrator rights. After work, remove any temporary CA additions as rollback.
5. If corporate TLS inspection is present, coordinate to distribute the inspection CA or adjust proxy settings.

## Common mistakes (do not)
- Leaving certificate validation disabled.
- Permanently adding untrusted/self-signed CA to the Root store.

## Recurrence checklist
- Automate renewal (ACME/scheduler).
- Configure alerting and on-call rotation for expiry.
- Enforce NTP synchronization.
- Always deploy full chain with certificates.

## Related cases
- Missing intermediate CA prevents connection.
- Large client clock skew causing validation errors.
- Proxy/TLS inspection replacing server certs.

## References
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/en-us/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/en-us/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/en-us/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult