---
title: "Microsoft Store アプリのインストールに失敗 0x80073CF9"
errorText: "Microsoft Store アプリのインストールに失敗 0x80073CF9"
tags: ["Windows", "Store", "App"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  Microsoft Store または UWP パッケージのインストール時にエラー 0x80073CF9 が発生し、インストールが中断される。  
- 影響（1行）  
  アプリがインストールできず、既存の更新や修復も失敗する。  
- 緊急度（中、理由1行）  
  インストール作業の阻害で業務に影響するが、システム全体の即時停止を招くわけではないため中。

## よくある原因（優先度順）
1. パッケージ格納先またはTEMPフォルダーのアクセス許可不足（所有権/ACLが不適切）  
2. Microsoft Defender や EDR によるブロック（制御されたフォルダーアクセスや検疫）  
3. 不完全なダウンロードやブロックされた Zone.Identifier（MOTW）による実行制限  
4. ファイル/ハンドルが別プロセスにロックされている（インストーラーの一時ファイルが使用中）  
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
1) 対象を特定 → 所有権/権限を付与（GUI またはコマンドで限定的に実施）  
2) 制御されたフォルダーアクセスやEDRの許可登録、MOTW(Zone.Identifier)の解除を実施  
   - PowerShell（ブロック解除）:
     ```powershell
     Unblock-File -Path "C:\Path\to\installer-or-script.exe"
     ```
   - GUI: 「プロパティ → ブロックの解除」にチェック  
   - Defender 許可追加（例）:
     ```powershell
     Add-MpPreference -ControlledFolderAccessAllowedApplications "C:\Path\app.exe"
     ```
3) ロック中ならリブート／セーフモード／保持プロセスの安全な終了（resmon でハンドル検索）

## 根本対策（恒久策）
- 所有権と継承の是正（影響範囲を限定して実施）  
- Defender/EDR: 許可アプリ/例外ポリシーを整備し運用ルール化  
- GPO/EDR配下での申請フローを整備し、インストーラー配布時に例外を事前登録  
- 共有やZIPから直接実行せず、ローカル（例: C:\Temp）にコピーしてから実行

## 詳細手順（Windows 10/11）
1. GUIでの設定変更：対象フォルダーのプロパティ → セキュリティ → 詳細設定 → 所有者変更／継承設定の確認。  
2. コマンド例（管理者で実行）
   ```cmd
   takeown /F "C:\Path\To\Folder" /R /D Y
   icacls "C:\Path\To\Folder" /grant Administrators:F /T
   ```
   ⚠ 注意: 所有権とACLを広範に変更する（ロールバック: 所有者を SYSTEM に戻す）。  
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
4. ファイルがロック中なら（見つけ方）  
   - Win + R → resmon → CPU タブ → 関連付けられたハンドルでパスを検索 → 該当プロセスを終了、または再起動／セーフモードで作業。  
⚠ 注意: 一時的な Defender/EDR 無効化は作業中のみ許可。完了後は必ず元に戻す（ロールバック: Defender 設定を元に戻す）。

## よくある落とし穴（やってはいけない）
- Windows\Program Files や System32 直下への広範な所有権変更を行う  
- 共有フォルダーに対して Everyone:フル を付与する

## 再発防止チェックリスト
- 標準ユーザー運用を原則とし、必要時のみ昇格を実施  
- インストーラーはダウンロード直後に実行してロックを回避  
- 変更手順を文書化し、変更レビューを実施

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