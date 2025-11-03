---
title: "Microsoft Defender がグループポリシーにより無効化"
errorText: "Microsoft Defender がグループポリシーにより無効化"
tags: ["Windows", "Defender", "Policy"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  グループポリシー（GPO）または管理用ポリシーにより Microsoft Defender Antivirus が無効化されているときに発生。
- 影響（1行）  
  エンドポイントのリアルタイム保護が停止し、マルウェア/ランサムウェア感染リスクが上昇する。
- 緊急度（高/中/低、理由1行）  
  高 — 保護が無効化された端末は即座に攻撃対象となるため、速やかな確認と対応が必要。

## よくある原因（優先度順）
1. 組織のGPOで「Windows Defender を無効にする」ポリシーが設定されている  
2. MDM（Intune）やEDR管理コンソールでの配布ポリシーにより無効化されている  
3. Tamper Protection によりローカルでの復帰操作がブロックされている  
4. ファイル/レジストリの権限不整合やロックによりサービスが開始できない（ファイルロックが原因）  
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen などの実行制御によるブロック

## 60秒で確認（最短手順）
- 管理者で起動 → 再試行  
- 現在のACLを確認：
  ```cmd
  icacls "C:\Path\To\Folder"
  ```
- 自分の特権を確認：
  ```cmd
  whoami /groups & whoami /priv
  ```
- Defenderのブロック/履歴とCFA有効状態を確認：
  ```powershell
  Get-MpThreatDetection -ErrorAction SilentlyContinue | Select Timestamp, Action, Resources | Select -First 5
  Get-MpPreference | Select EnableControlledFolderAccess
  ```
- TEMPフォルダーの権限を確認：
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  ```
- GPOの影響をスナップショット：
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5分で解決（実務フロー）
1) 影響端末を特定 → gpresult でどのGPOが適用されているか確認する。  
2) GPOが原因なら、ポリシー管理者に変更申請して組織方針を確認し、必要に応じてポリシーを緩和・無効化する。  
3) ローカルで回復が可能な場合は、レジストリとローカルポリシーを確認・編集する（管理者権限が必要）。  
   ⚠ 注意: レジストリ編集やポリシー変更は運用者の同意を得てから実行。完了後は元に戻す。  
   例（無効化フラグ確認）:
   ```cmd
   reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware
   ```
   例（ローカルで有効化する短期手段）:
   ```cmd
   reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiSpyware /t REG_DWORD /d 0 /f
   ```
   ロールバック: 変更前の値に戻すか、ポリシー側で再適用してもらう。  
4) Tamper Protection や EDR が原因なら管理コンソールで設定を変更する。  
5) Defender 設定の確認と許可登録（制御されたフォルダーアクセスや例外追加）:
   ```powershell
   Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
   ```
6) ファイルがロック中ならリブート/セーフモードで作業、または該当プロセスを安全に終了して再試行。

## 根本対策（恒久策）
- GPO/MDM/EDRの設定変更は公式な申請フローを確立し、変更履歴を残す。  
- Tamper Protection と EDR ポリシーの役割分担を明確化する。  
- 例外登録の承認基準を定め、最小権限での運用を維持する。  
- インストーラーや管理スクリプトはローカルに配置し（例: C:\Temp）、実行する運用ルールを作成する。

## 詳細手順（Windows 10/11）
1. GUIでの確認  
   - 「gpedit.msc」または管理用GPOで「Windows Defender Antivirus」を確認。  
   - セキュリティセンターで「Tamper Protection」を確認し、管理者が変更できるか確認。  
2. コマンド例（管理者で実行）
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   **ロールバック例**（SYSTEMに戻す）:
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   **付け過ぎた権限を既定に戻す（継承/ACLの修復）**:
   ```cmd
   icacls "C:\Path\To\Folder" /inheritance:e
   icacls "C:\Path\To\Folder" /reset
   ```
3. Defender: 状態確認と許可追加
   ```powershell
   Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
   ```
4. ファイルがロック中なら（見つけ方）
   - 「Win + R」→「resmon」→ CPU タブ → 関連付けられたハンドルでパスを検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: 一時的に Defender を無効化する操作は作業時間を限定し、完了後に必ず再有効化。ロールバックは元のポリシー値に戻す。

## よくある落とし穴（やってはいけない）
- Windows/Program Files/System32 直下への広範な所有権変更。  
- 共有フォルダーに対して Everyone:Full を付与すること。  
- 管理者権限なしで Tamper Protection を無効化しようと繰り返すこと。

## 再発防止チェックリスト
- 標準ユーザー運用＋必要時のみ昇格を徹底する。  
- ポリシー変更は変更管理プロセスで承認・記録する。  
- インストール手順を文書化し、ローカルで実行する運用にする。  
- 定期的に gpresult を使ってGPO適用状況を監査する。

## 関連ケース
- Windows Update で 0x80070005 が出るときの手順  
- Office/Outlook の有効化で 0x80070005 が出るとき  
- OneDrive/同期クライアントで 0x80070005 が出るとき

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult