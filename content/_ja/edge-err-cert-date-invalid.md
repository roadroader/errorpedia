---
title: "この接続ではプライバシーが保護されません（証明書の有効期限切れ）"
errorText: "この接続ではプライバシーが保護されません（証明書の有効期限切れ）"
tags: ["Edge", "Certificate", "HTTPS"]
updatedAt: "2025-11-03T09:11:56.107Z"
lang: "ja"
---

## 要約
- 発生条件（1行）  
  ブラウザーがアクセス先サーバーのTLS証明書の有効期限を検出し、接続を遮断したときに表示されるエラー。
- 影響（1行）  
  該当サイトへのHTTPSアクセスが拒否され、機密データ送受信ができない。
- 緊急度（高、理由1行）  
  高：中間者攻撃や証明書運用停止を示す可能性があり、業務サービス停止につながるため即対応を推奨。

## よくある原因（優先度順）
1. サーバー側の証明書が期限切れ（運用者未更新）。
2. クライアント端末の時刻が大きくずれている（検証に失敗）。
3. 中間CAまたはルート証明書チェーンが欠落/失効。
4. ファイルロック／ストアの書き込み不能（ローカル証明書キャッシュや証明書ストアがロックされ更新できない）。
5. AppLocker / ソフトウェア制限ポリシー(SRP) / SmartScreen などの実行制御によるブロック（証明書チェーン確認ツールやエージェントが動作しない）。

## 60秒で確認（最短手順）
- 管理者でブラウザーを起動 → 同じURLを再試行してエラー詳細を確認（証明書の有効期間を確認）。
- システム時刻を確認・同期：
  ```cmd
  w32tm /query /status
  ```
- 証明書をエクスポートして期限を確認（ブラウザーの詳細表示からエクスポート）。
- エクスポートした証明書の検証（簡易）：
  ```cmd
  certutil -verify "C:\Path\to\exported-cert.cer"
  ```
- 現在のACLを確認：
  ```cmd
  icacls "C:\Windows\System32\CertLog" 
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
1) クライアント側確認：時刻同期を行い、ブラウザーの証明書詳細で「発行日/有効期限」「サブジェクト」「サムプリント」を確認。  
2) サーバー運用者へ連絡：証明書の更新（再発行・デプロイ）を依頼。期限切れなら即更新を依頼。  
3) 一時回避（慎重に）：
   - ブラウザーの「詳細設定/続行（安全でない可能性あり）」を使うことは避けるが、業務上どうしても必要な場合のみ実施。  
   ⚠ 注意: ブラウザーで例外を追加するのは危険。作業完了後すぐに例外を削除して元に戻す。
   - 一時的にコマンドラインで検査する例（証明書チェーン取得）：
     ```cmd
     openssl s_client -connect example.com:443 -servername example.com -showcerts
     ```
4) 中間CA/ルートの欠落なら、適切な中間証明書をサーバーに追加またはクライアントの信頼ストアへインストール（管理者権限必要）。  
   ⚠ 注意: 信頼ストアへ自己署名/不明なCAを追加するのは危険。追加後は必ず元に戻す（ロールバック手順を下記に記載）。
5) ネットワーク経路（プロキシ/TLSインスペクション）を確認し、社内プロキシが独自証明書で中間者していないか確認。

## 根本対策（恒久策）
- 自動更新（ACME/スケジューラ）で証明書の期限管理を自動化する。
- 監視：有効期限切れのアラートを運用監視に組み込む。
- 時刻同期（NTP）を全端末で強制適用。
- 中間CAチェーンをサーバーに完全にインストールしておく。
- 証明書発行・更新の手順をドキュメント化し、担当者を明確にする。

## 詳細手順（Windows 10/11）
1. ブラウザーで証明書を表示 → 「証明書のパス」からチェーンを確認。中間証明書が表示されない場合はサーバー構成を修正する。  
2. ローカルで証明書を一時インストール（管理者）：
   ```cmd
   certutil -addstore "Root" "C:\Path\to\root-ca.cer"
   ```
   **ロールバック例（Rootから削除）**:
   ```cmd
   certutil -delstore "Root" "THUMBPRINT_OF_CERT"
   ```
3. 証明書の検証（コマンド例）:
   ```cmd
   certutil -verify "C:\Path\to\exported-cert.cer"
   ```
4. ファイルがロック中なら（ストアの更新ができない場合）
   - 再起動して保有プロセスを解放、またはセーフモードで操作。  
   ⚠ 注意: 証明書ストア操作は管理者のみ実施。完了後は必ず不要なCAを削除してロールバックする。
5. 必要に応じてネットワーク/プロキシ担当と協力し、TLS中間者の証明書を配布するか、プロキシ設定を調整する。

## よくある落とし穴（やってはいけない）
- 一時的に「証明書検証を無効化」したまま運用すること。
- 信頼できない自己署名証明書をルートストアに恒久的に追加すること。

## 再発防止チェックリスト
- 期限管理の自動化（ACME/スケジュール）を導入。
- 監視アラートと担当者のローテーションを設定。
- NTP同期の強制適用。
- サーバーに中間チェーンを常時含めてデプロイ。

## 関連ケース
- 中間CAが失効して接続できないケース
- クライアント時刻が大きくずれているケース
- プロキシ/TLSインスペクションによる証明書置換

## 参考（一次情報）
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/icacls
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/takeown
- https://learn.microsoft.com/ja-jp/powershell/module/defender/get-mppreference
- https://learn.microsoft.com/ja-jp/powershell/module/defender/add-mppreference
- https://learn.microsoft.com/ja-jp/defender-endpoint/enable-controlled-folders
- https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/gpresult