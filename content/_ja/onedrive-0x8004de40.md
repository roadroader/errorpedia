---
title: "OneDrive にサインインできません 0x8004DE40"
errorText: "OneDrive にサインインできません 0x8004DE40"
tags: ["OneDrive", "Network", "Proxy"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）
  - OneDrive クライアントでサインインを試行した際にネットワーク/資格情報またはローカル権限の不整合で発生するエラー（コード: 0x8004DE40）。
- 影響（1行）
  - サインイン不可により同期が止まり、ユーザーデータや設定がクラウドと同期されない。
- 緊急度（中、理由1行）
  - 中。個別ユーザー回復は迅速だが、企業環境では多人数に波及し得るため優先対応が必要。

## よくある原因（優先度順）
1. ネットワーク/プロキシ設定で OneDrive の認証先に到達できない（DNS、プロキシ、FW）。
2. 保存された資格情報の破損やブラウザの認証キャッシュ不整合。
3. OneDrive アプリの構成ファイルやキャッシュの破損（再インストールで解消する場合あり）。
4. ファイル/フォルダーが別プロセスにロックされており認証フローが失敗するケース。
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen / EDR による実行制御で接続やプロセス起動がブロックされる。

## 60秒で確認（最短手順）
- 管理者で起動 → OneDrive を再試行
- 現在のACLを確認：
  ```cmd
  icacls "C:\Users\%USERNAME%\AppData\Local\Microsoft\OneDrive"
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
1) 対象を特定 → 所有権/権限を必要最小限で付与（GUI またはコマンド）
2) 制御されたフォルダーアクセスやEDRの一時的な許可登録、MOTW(Zone.Identifier)の解除を実施
   - PowerShell（ブロック解除）:
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: ダウンロードファイルの「プロパティ → ブロックの解除」にチェックして適用
3) ロック中ならプロセスを安全に終了、またはリブート／セーフモードで作業し再試行

⚠ 注意: Defender/EDR を完全無効化する手順は推奨しない。作業中のみ一時変更し、終了後速やかに元に戻す（ロールバック：設定を元に戻してから再検証）。

## 根本対策（恒久策）
- OneDrive 用フォルダーと実行ファイルの所有権・継承を局所的に正す（広範囲変更は避ける）。
- Defender/EDR に許可アプリや例外ポリシーを作成して誤検知を防止。
- GPO・AppLocker 配下の環境ではインストーラー/アプリ配布の承認フローを整備。
- 共有やZIP内から直接実行せず、ローカル（例: C:\Temp）にコピーしてから実行する運用を徹底。

## 詳細手順（Windows 10/11）
1. GUIでの設定変更
   - 対象フォルダーのプロパティ → セキュリティ → 詳細設定 → 所有者を変更 → 継承を有効化 → 必要最小権限を付与。
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
3. Defender: 設定と許可アプリ
   - 状態確認:
     ```powershell
     Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
     ```
   - 許可追加:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\OneDrive\OneDrive.exe"
     ```
4. ファイルがロック中なら（見つけ方）
   - Win + R → resmon → CPU タブ → 関連付けられたハンドルでパスを検索 → 該当プロセスを終了、または再起動／セーフモードで作業。

⚠ 注意: 所有権を広範に置き換えるとOS/アプリに影響が出る。ロールバック：問題なければ速やかに元の所有者に戻す。

## よくある落とし穴（やってはいけない）
- Windows/Program Files/System32 直下への広範な所有権変更。
- 共有フォルダーに Everyone:フル を付与してしまうこと。

## 再発防止チェックリスト
- 標準ユーザー運用を徹底し、必要時のみ昇格する手順を文書化。
- インストーラーはダウンロード直後に実行してブロック/ロックを回避。
- 変更手順（権限・例外登録）は承認フローとレビューを実施。

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