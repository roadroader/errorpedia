---
title: "Windows のライセンス認証に失敗 0xC004F074"
errorText: "Windows のライセンス認証に失敗 0xC004F074"
tags: ["Windows", "Activation", "KMS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  KMS/MAKによるライセンス認証が失敗し、エラーコード「0xC004F074（KMS サーバーが見つからない／応答なし）」が返る。
- 影響（1行）  
  Windows が未認証状態になり、機能制限や更新・ライセンスコンプライアンスに影響する。
- 緊急度（高、理由）  
  高（未認証は業務PCの機能制限・ライセンス違反につながるため速やかな対応が必要）。

## よくある原因（優先度順）
1. KMS サーバーへの到達不能（DNS SRV レコード不在、ネットワーク/ファイアウォール遮断）
2. プロダクトキーの種別不一致（KMSキーとMAKの混用、ボリュームライセンスの適用ミス）
3. 時刻同期・ドメイン参加の問題（クライアントとKMSホストで時刻差・ドメイン信頼エラー）
4. 一時ファイルやインストーラーがロックされ、認証プロセスが失敗するケース
5. AppLocker / SRP / SmartScreen 等の実行制御によるブロック

## 60秒で確認（最短手順）
- 管理者でコマンドプロンプトを起動 → 再試行
- ライセンス状態を確認：
  ```cmd
  slmgr /dli
  slmgr /dlv
  slmgr /xpr
  ```
- KMS 応答をテスト（ドメイン名を置換）：
  ```cmd
  nslookup -type=SRV _vlmcs._tcp.yourdomain.local
  ```
- 現在のACLを確認：
  ```cmd
  icacls "C:\\Path\\To\\Folder"
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
  gpresult /h "%USERPROFILE%\\Desktop\\gp.html"
  ```

## 5分で解決（実務フロー）
1) ネットワーク経路を開通 → KMS SRV レコードとポート（TCP 1688）を確認。到達すれば再認証を実行：
   ```cmd
   slmgr /ato
   ```
2) キー種別を確認 → 必要であれば正しいキーを適用：
   ```cmd
   slmgr /ipk <Your-Product-Key>
   slmgr /ato
   ```
3) 時刻同期が原因なら同期実行：
   ```cmd
   w32tm /resync
   ```
4) Defender/EDRがブロックしている場合は許可登録または一時除外を実施（後で戻す）：
   ```powershell
   Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\\Windows\\System32\\slmgr.vbs"
   ```
5) ファイルロックがあるなら該当プロセスを終了するか再起動／セーフモードで実行。

⚠ 注意: ファイアウォールやEDRを無効化するのは一時的に限定すること。戻し方（ロールバック）: 無効化した設定を直ちに元に戻す（例: Firewall を有効化、EDR ポリシーを再適用）。

## 根本対策（恒久策）
- KMSインフラの監視（SRVレコード/応答時間/ポート監視）を導入。
- ライセンス割当フローを定義し、MAK/KMSの適用手順を標準化。
- NTP/時刻同期の監査と自動修正を実装。
- Defender/EDRの例外ポリシーを整備し、認証プロセスを除外リストに入れる。
- インストーラーはローカルに保存してから実行し、ファイルロックを避ける。

## 詳細手順（Windows 10/11）
1. GUIでの確認  
   - 設定 → 更新とセキュリティ → ライセンス認証 を開き、エラー詳細を確認。  
   - サービス.msc で "Software Protection"（sppsvc）を確認し、起動しているかを確認。停止した場合は開始。
2. コマンド例（管理者で実行）
   ```cmd
   slmgr /dli
   slmgr /dlv
   slmgr /xpr
   slmgr /ipk <Product-Key>
   slmgr /ato
   nslookup -type=SRV _vlmcs._tcp.yourdomain.local
   ```
   ⚠ 注意: sppsvc を停止して認証を試すのは推奨されない（サービス停止は一時的に機能を阻害）。戻し方（ロールバック）:
   ```cmd
   sc config sppsvc start= auto
   net start sppsvc
   ```
3. 所有権/権限の修正（インストーラーがアクセス権で失敗する場合）
   ```cmd
   takeown /F "C:\\Path\\To\\Folder" /R /D Y
   icacls "C:\\Path\\To\\Folder" /grant Administrators:F /T
   ```
   ロールバック例（SYSTEMに戻す）:
   ```powershell
   $p = "C:\\Path\\To\\Folder"
   icacls $p /setowner "NT AUTHORITY\\SYSTEM" /T
   ```
   付け過ぎた権限を既定に戻す（継承/ACLの修復）:
   ```cmd
   icacls "C:\\Path\\To\\Folder" /inheritance:e
   icacls "C:\\Path\\To\\Folder" /reset
   ```
4. Defender: 設定と許可アプリ
   - 状態確認:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - 許可追加:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\\Windows\\System32\\slmgr.vbs"
     ```
5. ロック中のファイルを特定する方法  
   - 「Win + R」→「resmon」→ CPU タブ → 関連付けられたハンドルでパス検索 → 該当プロセスを終了、または再起動／セーフモードで作業。

⚠ 注意: 一時無効化は作業中のみ。完了後は必ず元に戻す。

## よくある落とし穴（やってはいけない）
- ドメイン環境でKMS/MAKを混在させて無計画にキーを配布すること。
- Firewall に全開のポート開放を恒久的に行うこと。
- Windows/System32 等に対する広範な所有権変更。

## 再発防止チェックリスト
- KMS サーバーの可用性監視を設定する。
- NTP 同期の正常性を監査する。
- EDR/Defender の例外ポリシーをドキュメント化する。
- ライセンス配布・更新の手順を運用フローとして整備する。

## 関連ケース
- KMS ホスト移行後に 0xC004F074 が出るケース
- ネットワーク分離されたクライアントでの認証タイムアウト
- 時刻/ドメイン不整合による認証失敗

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult