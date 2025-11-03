---
title: "PowerShell スクリプトが実行ポリシーでブロック"
errorText: "PowerShell スクリプトが実行ポリシーでブロック"
tags: ["PowerShell", "Policy", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件：PowerShell スクリプトを実行した際に「スクリプトが実行ポリシーでブロックされました」等のエラーが出る（ローカル/ドメインポリシーやDefenderによるブロックが原因）。  
- 影響：スクリプト実行が停止し、インストール・構成・自動化処理が完了しない。  
- 緊急度：中（ユーザー作業は止まるが、通常は特権/ポリシー変更で短時間で復旧可能）。

## よくある原因（優先度順）
1. ローカルまたはマシン/ユーザースコープの ExecutionPolicy が Restricted/AllSigned 等で制限されている。  
2. ドメインGPOで実行ポリシーが強制されている（管理者権限でも上書き不可）。  
3. ファイルに Zone.Identifier (インターネットからのダウンロードフラグ) が付与され、Unblock が必要。  
4. ファイルが別プロセスにロックされているため、ヘルプテキストや署名検査が失敗することがある（ロックは実行失敗を引き起こす）。  
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen / EDR の実行制御でブロックされている。

## 60秒で確認（最短手順）
- 管理者で PowerShell を起動 → 再試行。  
- 現在の ACL を確認：
```cmd
icacls "C:\Path\To\Folder"
```
- 自分の特権を確認：
```cmd
whoami /groups & whoami /priv
```
- Defender のブロック/履歴と CFA 有効状態を確認：
```powershell
Get-MpThreatDetection -ErrorAction SilentlyContinue | Select Timestamp, Action, Resources | Select -First 5
Get-MpPreference | Select EnableControlledFolderAccess
```
- TEMP フォルダーの権限を確認：
```cmd
echo %TEMP%
icacls "%TEMP%"
```
- GPO の影響をスナップショット：
```cmd
gpresult /h "%USERPROFILE%\Desktop\gp.html"
```

## 5分で解決（実務フロー）
1) 実行対象スクリプトの署名/Zone 情報を確認し、ダウンロード品ならまずローカルにコピー。  
2) 必要なら Zone.Identifier を解除または Unblock-File を実施：
```powershell
Unblock-File -Path "C:\Path\to\script.ps1"
```
3) 一時的にプロセススコープで実行ポリシーを緩めて検証（管理者で実行）:
⚠ 注意: 永続的に実行ポリシーを緩めるとセキュリティリスク。作業後はセッションを閉じて元に戻す（Process スコープはセッション終了で戻る）。
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\script.ps1
```
4) Defender/EDR が原因なら許可登録（CFA 例）または EDR 管理コンソールで例外化。完了後は例外を最小化。

## 根本対策（恒久策）
- ExecutionPolicy を運用ポリシーに沿ってドキュメント化し、GPO で一貫性を持たせる。  
- スクリプトに適切な署名を付与し、信頼済みリポジトリ/配置場所を定義する。  
- Defender/EDR の例外ルールと申請フローを整備する。  
- 共有や ZIP 直開きでの実行を避け、必ずローカル（例: C:\Temp）に展開してから実行する。

## 詳細手順（Windows 10/11）
1. GUI での確認：スクリプトのプロパティで「ブロックの解除」にチェックし、署名状態を確認。  
2. コマンド例（管理者で実行）:
```cmd
takeown /F "C:\Path\To\Folder" /R /D Y
icacls "C:\Path\To\Folder" /grant Administrators:F /T
```
ロールバック例（SYSTEM に戻す）:
```powershell
$p = "C:\Path\To\Folder"
icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
```
付け過ぎた権限を既定に戻す（継承/ACL 修復）:
```cmd
icacls "C:\Path\To\Folder" /inheritance:e
icacls "C:\Path\To\Folder" /reset
```
3. Defender: 状態確認と許可追加
```powershell
Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
```
4. ファイルがロック中なら（見つけ方）:
- Win + R → resmon → CPU → Associated Handles でパス検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: プロセスを強制終了する前に影響範囲を確認すること。終了後は必ずサービスを正常状態に戻す。

## よくある落とし穴（やってはいけない）
- Windows や Program Files/System32 直下に対して広範な所有権変更を行う。  
- 共有フォルダーに Everyone:Full を付与して問題を回避する。

## 再発防止チェックリスト
- 標準ユーザー運用を徹底し、必要時のみ昇格する手順を明記。  
- スクリプトを配布する際に署名と配置場所を統一する。  
- Defender/EDR 例外は申請ログを残す。  
- 変更手順を文書化しレビューを実施する。

## 関連ケース
- Windows Update で 0x80070005 が出るときの手順  
- Office/Outlook の有効化で 0x80070005 が出るとき  
- OneDrive/同期クライアントで 0x80070005 が出るとき

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls