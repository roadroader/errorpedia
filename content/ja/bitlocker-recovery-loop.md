---
title: "BitLocker が毎回回復キーを要求する"
errorText: "BitLocker が毎回回復キーを要求する"
tags: ["Windows", "BitLocker", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  BitLocker 回復キーのプロンプトが毎回表示され、通常のサインインで暗号化ドライブがロックされる。  
- 影響（1行）  
  ユーザーの業務中断、リモートデバイスの自動復旧失敗、および運用コスト増。  
- 緊急度（高/中/低、理由1行）  
  高 — 作業不能に直結し、企業環境では運用停止やキー管理の漏れリスクがあるため。

## よくある原因（優先度順）
1. TPM と OS 間の状態不一致（ファームウェア更新やBIOS/UEFI設定変更）。  
2. Boot構成やSecure Bootの変更、ブートローダー書き換え。  
3. グループポリシー／キー管理（MBAM/Intune）での誤設定や同期エラー。  
4. ファイルロックやドライブアクセス失敗（起動時に必要ファイルへアクセスできない）。  
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen などの実行制御によるブロック。

## 60秒で確認（最短手順）
- 管理者で起動して再試行。  
- 現在のACLを確認：
  ```cmd
  icacls "C:\Windows\System32"
  ```
- 自分の特権を確認：
  ```cmd
  whoami /groups & whoami /priv
  ```
- Defenderの設定（CFA含む）を確認：
  ```powershell
  Get-MpPreference | Select EnableControlledFolderAccess, ControlledFolderAccessAllowedApplications
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
1) 発生対象デバイスを特定し、最近のファームウェア/BIOS更新とGPO変更を確認。  
2) 所有権/権限が不足している場合は一時付与（下記コマンド参照）。  
3) 制御されたフォルダーアクセスやEDRでブロックされている実行ファイルを許可登録、またはインストール元のZone情報を解除して再試行。  
   - PowerShell（ブロック解除）:
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: ファイルの「プロパティ → ブロックの解除」にチェック。  
4) ファイルがロック中なら関連プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: 一時的にセキュリティ機能（CFA/EDR）を無効化する場合は作業時間を限定し、完了後すぐに元に戻す。戻し方：無効化した設定を逆手順で再有効化する。

## 根本対策（恒久策）
- TPMファームウェアとUEFI設定の変更管理を運用ルール化。  
- BitLocker回復キーをAD/Intuneに一元保管し、キー同期の監視を設定。  
- Defender/EDRで許可アプリや例外ポリシーを整備。  
- 共有や圧縮ファイル直実行を避け、ローカルにコピーしてから実行する運用に変更。

## 詳細手順（Windows 10/11）
1. GUIでの権限修正：対象フォルダー→プロパティ→セキュリティ→詳細設定→所有者変更→継承設定を確認。  
2. コマンド例（管理者で実行）  
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   ロールバック例（SYSTEMに戻す）:
   ```powershell
   $p = "C:\Path\To\Folder"
   icacls $p /setowner "NT AUTHORITY\SYSTEM" /T
   ```
   付け過ぎた権限を既定に戻す（継承/ACLの修復）:
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
   - Win + R → resmon → CPU タブ → 関連付けられたハンドルでパス検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: システム領域の広範な所有権変更は避ける。ロールバック：所有者をSYSTEMに戻し、ACLをリセットする。

## よくある落とし穴（やってはいけない）
- Windows/Program Files/System32 直下への広範な所有権変更。  
- 共有フォルダーで Everyone:フル を付与。  
- セキュリティ無効化を長時間継続する。

## 再発防止チェックリスト
- 標準ユーザー運用を徹底し、必要時のみ昇格。  
- BitLockerキーの中央管理と定期バックアップ。  
- 変更手順の文書化と変更管理（ファームウェア/BIOS含む）。  
- インストーラーはダウンロード直後に実行し、ZIP/ネットワーク直実行を避ける。

## 関連ケース
- Windows Updateで0x80070005が出るときの手順。  
- Office/Outlookの有効化で0x80070005が出るとき。  
- OneDrive/同期クライアントで0x80070005が出るとき。

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference  
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference  
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders  
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult