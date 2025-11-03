---
title: "プリンターに接続できません 0x0000011b"
errorText: "プリンターに接続できません 0x0000011b"
tags: ["Windows", "Printer", "Network"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  ネットワーク/共有プリンターへ接続時に Windows がプリントジョブを送信できずエラーコード「0x0000011b」を返す。  
- 影響（1行）  
  プリンター接続不可、印刷失敗、共有プリンターを使う業務が停止する。  
- 緊急度（中、認証/共有/権限の不整合で業務復旧が必要なため）

## よくある原因（優先度順）
1. クライアントとプリントサーバー間のSMB認証/署名ポリシー不一致（Windows更新後に発生しやすい）。  
2. Print Spoolerサービスの権限/プロセス障害（サービスが応答しない・アクセス拒否）。  
3. ネットワーク経路/名前解決の問題（DNS、ホスト名解決が失敗）。  
4. ファイル/ポートがロックされている（プリントスプールファイルや一時ファイルが別プロセスに占有）。  
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen などの実行制御によるブロック

## 60秒で確認（最短手順）
- 管理者でコマンドプロンプトを起動 → 再試行  
- Print Spooler の状態確認と再起動：
  ```cmd
  sc query spooler
  sc stop spooler
  sc start spooler
  ```
- 現在のACLを確認：
  ```cmd
  icacls "C:\Windows\System32\spool\PRINTERS"
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
- TEMP/スプール一時フォルダーの権限とロック確認：
  ```cmd
  echo %TEMP%
  icacls "%TEMP%"
  handle -a PRINTERS  (Sysinternals handle を利用する場合)
  ```
- GPOの影響をスナップショット：
  ```cmd
  gpresult /h "%USERPROFILE%\Desktop\gp.html"
  ```

## 5分で解決（実務フロー）
1) 対象を特定 → Print Spooler を再起動・ログ確認（イベントビューア：System/Application）。  
2) 所有権やACLが原因なら一時的に所有権を取得し権限を付与（管理者で実行）。  
   - コマンド例（管理者）:
     ```cmd
     takeown /F "C:\Windows\System32\spool\PRINTERS" /R /D Y
     icacls "C:\Windows\System32\spool\PRINTERS" /grant Administrators:F /T
     ```
   ⚠ 注意: 所有権を付与する操作はシステム挙動に影響するため作業後に元に戻すこと。ロールバック：上の「詳細手順」のロールバック例参照。  
3) Defender/EDRやControlled Folder Accessがスプールファイルやドライバーをブロックしていないか確認・許可登録。  
   - ブロック解除（PowerShell）:
     ```powershell
     Unblock-File -Path "C:\Path\to\printer-driver-installer.exe"
     ```
   - GUI: ドライバーやインストーラーの「プロパティ → ブロックの解除」にチェック  
4) ファイルロックが原因なら該当プロセスを終了、サービス再起動、または再起動／セーフモードで操作。

## 根本対策（恒久策）
- プリントサーバー側とクライアント側でSMB/認証ポリシーを整合させ、更新適用ポリシーを運用に組み込む。  
- Print Spoolerの権限とフォルダー継承を監視し、過剰な権限変更を禁止。  
- Defender/EDRでプリンタードライバーとスプールフォルダーを許可アプリに追加。  
- 共有プリンターを使う場合は名前解決の監視とIP直接接続の代替手順を用意。

## 詳細手順（Windows 10/11）
1. GUIでの設定変更（対象フォルダーの所有者を確認→継承を有効→管理者に必要最小権限を付与）。  
2. コマンド例（管理者で実行）
   ```cmd
   takeown /F "C:\Windows\System32\spool\PRINTERS" /R /D Y
   icacls "C:\Windows\System32\spool\PRINTERS" /grant Administrators:F /T
   ```
   **ロールバック例**（SYSTEMに戻す）:
   ```powershell
   $p = "C:\Windows\System32\spool\PRINTERS"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   **付け過ぎた権限を既定に戻す（継承/ACLの修復）**:
   ```cmd
   icacls "C:\Windows\System32\spool\PRINTERS" /inheritance:e
   icacls "C:\Windows\System32\spool\PRINTERS" /reset
   ```
3. Defender: 設定と許可アプリ  
   - 状態確認:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - 許可追加:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Windows\System32\spool\drivers\x64\3\printerdriver.dll"
     ```
4. ファイルがロック中なら（見つけ方）
   - 「Win + R」→ `resmon` → CPU タブ → 関連付けられたハンドルでパスを検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: Defender/EDRの一時無効化や所有権付与は作業中のみ行う。作業完了後は必ず元に戻す（ロールバック）。

## よくある落とし穴（やってはいけない）
- System フォルダー全体や Program Files に対して広範に所有権を変更する。  
- 共有フォルダーに対して Everyone:フル を付与する。

## 再発防止チェックリスト
- 更新配信後のプリント検証手順を運用化。  
- EDR/GPOでプリンタードライバー例外の申請フローを整備。  
- インストーラーはダウンロード後ローカル（例: C:\Temp）に置いて実行する。

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