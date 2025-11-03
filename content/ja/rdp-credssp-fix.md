---
title: "RDP 接続エラー: CredSSP 暗号化オラクルを修復"
errorText: "RDP 接続エラー: CredSSP 暗号化オラクルを修復"
tags: ["Windows", "RDP", "Security"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）
  - RDPで「暗号化オラクルの修復 (CredSSP)」エラーが出て接続できない（クライアントとサーバーでCredSSPパッチ/設定が不整合）。
- 影響（1行）
  - リモートデスクトップ接続が失敗し、管理作業やアプリ操作が中断する。
- 緊急度（高、理由1行）
  - 高（リモート管理できないと復旧作業やセキュリティ修正が遅延するため）。

## よくある原因（優先度順）
1. クライアント/サーバーのWindows更新が揃っていない（CredSSP修正が片方だけ適用）。
2. ローカル/ドメインGPOで「Encryption Oracle Remediation」が厳格に設定されて接続拒否。
3. 証明書や資格情報の破損（NLAや資格情報キャッシュの不整合）。
4. ファイル/プロセスロックで古いRDP関連バイナリを置換できない（更新インストーラーが失敗）。
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen などの実行制御によるブロック。

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
- CredSSP 設定を確認：
  ```cmd
  reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle
  ```

## 5分で解決（実務フロー）
1) 最優先：クライアントと対象サーバーに最新のWindows更新を適用し再起動。これが根本解決。
2) 当面の回避（パッチ適用が不可の場合）：
   - ローカルで一時的にポリシーを緩める（リスクあり）：
     ```cmd
     reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 2 /f
     gpupdate /force
     ```
   - GUI: gpedit.msc → Computer Configuration → Administrative Templates → System → Credentials Delegation → Encryption Oracle Remediation を「Vulnerable」に設定。
   ⚠ 注意: この変更はセキュリティを低下させる。必ずパッチ適用後に元に戻す（ロールバック：下記参照）。
3) RDPサービス再起動/接続先再起動後に再試行：
   ```cmd
   net stop TermService & net start TermService
   ```
4) ロックが疑われる場合は対象プロセスを終了、または再起動/セーフモードでアップデート実行。

ロールバック（元に戻す一言）:
- パッチ適用後、次で値を戻す：
  ```cmd
  reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 0 /f
  gpupdate /force
  ```

## 根本対策（恒久策）
- 組織で定期的にWindows更新を同期的に展開（クライアント・サーバー双方を同時適用）。
- GPOテンプレートに変更管理と稼働前検証を導入。
- リモート管理用ジャンプサーバーを最新の状態で維持し、古いホストへ直接依存しない運用にする。
- インストーラーや管理ツールはローカルへコピーして実行（ネットワーク/共有直実行を避ける）。

## 詳細手順（Windows 10/11）
1. GUIでの確認
   - gpedit.msc / グループポリシー管理で Encryption Oracle Remediation の設定を確認。ドメイン環境ではGPO変更はGPO管理者に依頼。
2. コマンド例（管理者で実行）
   ```cmd
   reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle
   reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 2 /f
   gpupdate /force
   ```
   **ロールバック例**（安全に戻す）:
   ```cmd
   reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /v AllowEncryptionOracle /t REG_DWORD /d 0 /f
   gpupdate /force
   ```
3. 更新の確認と再試行
   - 両端末でWindows Update履歴を確認し、関連KBを適用して再起動。
4. ファイルがロック中なら（見つけ方）
   - リソースモニター（resmon）でハンドル検索 → 該当プロセス終了または再起動。

⚠ 注意: 一時的にポリシーを「Vulnerable」にするのは最終手段。パッチ適用後に必ず元に戻す。

## よくある落とし穴（やってはいけない）
- 永続的にAllowEncryptionOracleを「2（Vulnerable）」に設定して運用する。
- サーバー側を更新せずクライアントだけ対応して放置する。

## 再発防止チェックリスト
- 更新同期方針を定義（重要サーバーとクライアント群を同一週でロールアウト）。
- GPO変更はチケット/レビュー必須にする。
- 事前に検証用環境でCredSSP更新を適用して問題を洗い出す。

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