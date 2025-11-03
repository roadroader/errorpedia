---
title: "VPN 接続エラー 809 (L2TP/IPsec)"
errorText: "VPN 接続エラー 809 (L2TP/IPsec)"
tags: ["Windows", "VPN", "Networking"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）: L2TP/IPsec 接続で IPsec ネゴシエーションに失敗し、エラー 809 が発生する（NAT、ポート、サービス、証明書の問題が主因）。  
- 影響（1行）: リモートアクセスやサイト間VPNが確立できず業務システムにアクセス不可になる。  
- 緊急度（高/中/低、理由1行）: 高 — 通常業務に直結するため、短時間で復旧が必要。

## よくある原因（優先度順）
1. UDP 500/4500 ポートや ESP(プロトコル50) がネットワーク/ファイアウォールでブロックされている。  
2. サーバー側で IKE/IPsec サービス（RasMan / IKEEXT）が起動していない、またはL2TPが有効でない。  
3. 事前共有鍵(PSK)や証明書の不一致・期限切れ・CA未信頼。NATトラバーサル（NAT-T）未対応も含む。  
4. ファイルロックによりクライアント側の VPN バイナリ/一時ファイルが読み込めない（TEMP の権限不整合やプロセス保持）。  
5. AppLocker / SRP / SmartScreen 等の実行制御により rasphone.exe 等がブロックされる。

## 60秒で確認（最短手順）
- 管理者で起動 → 再試行  
- サービス状態確認（RasMan, IKEEXT）：
  ```powershell
  Get-Service RasMan, IKEEXT
  ```
- ポート到達性確認（VPN サーバーの IP を指定）：
  ```powershell
  Test-NetConnection -ComputerName vpn.example.com -Port 500
  Test-NetConnection -ComputerName vpn.example.com -Port 4500
  ```
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
- イベントログ確認（System, Application, RasClient, IKEEXT）:
  - 開く: イベント ビューア → Windows ログ → 該当ログを確認

## 5分で解決（実務フロー）
1) ポート/プロトコルを開放 → テスト  
   - ファイアウォールやルーターで UDP 500, UDP 4500、ESP を許可。NAT デバイスは NAT-T を有効化。  
2) サービスを起動 → 再接続  
   ```powershell
   Start-Service RasMan, IKEEXT
   ```
3) 認証情報を確認 → 再設定  
   - サーバーとクライアントの PSK または証明書を突き合わせる。  
4) クライアント側のアクセス権/ロックを解除  
   - 一時ファイルの権限を修正、該当プロセスを終了、または再起動。  
5) 制御されたフォルダーアクセスや EDR の一時許可登録、MOTW の解除
   - PowerShell（ブロック解除）:
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: 「プロパティ → ブロックの解除」にチェック

## 根本対策（恒久策）
- ネットワーク設計で必要ポート/プロトコルの明文化と監視導入。  
- サーバー側の証明書管理と自動更新（有効期限切れ防止）。  
- EDR/GPOで VPN 実行ファイルを許可リスト化、申請フロー化。  
- 共有メディア上でのインストーラー実行を避け、ローカルにコピーしてから実行。

## 詳細手順（Windows 10/11）
1. GUI での確認と変更（サービス、イベント、証明書ストア、VPN プロファイルのプロパティ）。  
2. コマンド例（管理者で実行） — 所有権/権限修正（必要時）:
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
   - 状態確認:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - 許可追加:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Windows\System32\rasphone.exe"
     ```
4. ファイルがロック中なら（見つけ方）
   - Win + R → resmon → CPU タブ → Associated Handles にパスで検索 → 該当プロセス終了、または再起動／セーフモードで作業。  
⚠ 注意: 一時的にファイアウォールや EDR を無効化して原因切り分けを行う場合は「作業中のみ」実施し、完了後は即座に元に戻す。ロールバック: 設定変更を元に戻す手順をメモしてから実行。

## よくある落とし穴（やってはいけない）
- VPN 関連サービスを恒久的に無効化する（デバッグ以外）。  
- サーバー/クライアントで広範に Everyone:Full を付与する。  
- 証明書の検証を恒久的に無効化する（pinning/検証を無効にしない）。

## 再発防止チェックリスト
- サーバー証明書の有効期限監視を設定。  
- ネットワーク機器で NAT-T 対応とログを有効化。  
- 変更手順を文書化し、テスト環境で検証。  
- EDR/GPO の許可ルールを定期レビュー。

## 関連ケース
- IPSec ネゴシエーション失敗で 13876/13894 系ログが出る場合の診断  
- L2TP クライアントで 691 等の認証エラーと併発するケース  
- NAT 環境でパケット変換により接続が不安定なケース

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult