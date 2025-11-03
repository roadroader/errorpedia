---
title: "Hyper-V を有効化できない（仮想化サポート無効）"
errorText: "Hyper-V を有効化できない（仮想化サポート無効）"
tags: ["Windows", "Hyper-V", "BIOS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  Hyper-V を有効化しようとすると「仮想化サポート無効」や BIOS/UEFI で無効の旨が表示され、有効化できない。  
- 影響（1行）  
  仮想マシンの作成・起動や WSL2 / Docker Desktop の利用ができない。  
- 緊急度（高/中/低、理由1行）  
  高 — 仮想化を要する開発・テスト作業やコンテナ環境が停止するため、即時対応が必要。

## よくある原因（優先度順）
1. BIOS/UEFI の仮想化（Intel VT-x / AMD-V）が無効  
2. OS エディションやセキュリティ機能（Windows Home / Core Isolation / Memory Integrity / Device Guard）がブロック  
3. Hyper-V 必須機能がインストールされていないまたは壊れている（Windows の機能）  
4. ファイルやドライバーがロック/アクセス拒否でインストールが失敗（インストーラー実行権限不足）  
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
1) BIOS/UEFI を確認：再起動 → ファームウェア設定（F2/Del 等） → 「Intel Virtualization Technology」「VT-x」「SVM」等を有効化 → 保存して再起動。  
2) OS・エディション確認：Windows Home では Hyper-V が利用不可（代替：WSL2/Docker Desktop の要件確認または Pro へのアップグレード）。  
3) Windows の機能をオンにする（管理者コマンド）:
   ```cmd
   dism /Online /Enable-Feature /All /FeatureName:Microsoft-Hyper-V
   ```
4) セキュリティによるブロック解除：Core Isolation / Memory Integrity を一時オフにして再試行（設定後は再起動が必要）。  
   ⚠ 注意: Memory Integrity を無効にするのは診断時のみ。作業完了後は必ず元に戻す（ロールバック：同じ設定画面で有効化）。  
5) インストーラーやスクリプトがブロックされている場合は許可登録またはブロック解除を実施。再起動／セーフモードでインストールを試す。

## 根本対策（恒久策）
- ファームウェア管理手順を標準化して仮想化を常時有効化（BIOS 設定管理）  
- Defender/EDR の許可アプリリストを整備し Hyper-V 及び関連インストーラーを登録  
- GPO/MDM で仮想化に関連するポリシー（Device Guard, Credential Guard）を明確化・申請フロー化  
- インストーラーはローカル（例: C:\Temp）にコピーしてから実行し、ZIP/共有から直接実行しない

## 詳細手順（Windows 10/11）
1. BIOS/UEFI の確認と有効化（手順は機種依存、必ずベンダー手順に従う）。  
2. コマンド例（管理者で実行、権限周りの修復）:
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
3. Defender: 設定と許可アプリ  
   - 状態確認:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - 許可追加:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
4. ファイルがロック中なら（見つけ方・対処）  
   - Win + R → resmon → CPU → 関連付けられたハンドルでパスを検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
   ⚠ 注意: プロセス強制終了は影響が大きい場合がある。終了前に影響範囲を確認し、完了後は必要に応じてサービスを再起動（ロールバック：終了したサービスを再起動）。

## よくある落とし穴（やってはいけない）
- Windows/Program Files/System32 直下への広範な所有権変更  
- 共有フォルダーの Everyone:フル 付与

## 再発防止チェックリスト
- 標準ユーザー運用＋必要時のみ昇格を徹底する  
- インストーラーはダウンロード直後にローカルへ展開して実行する  
- ファームウェア設定変更を構成管理ツールで追跡する

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