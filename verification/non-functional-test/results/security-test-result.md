# 非機能試験（セキュリティ）結果報告書

## ステータス
一部実施済み。S-09（Security Hub 検出結果レビュー）・S-07（WAF マネージドルールの有効性）・S-05（脅威検知・監査証跡の実効性）は合格。S-04 の一部（ALB 直アクセス 403）は確認済み。S-01（依存パッケージスキャン）は2026-09-26に実施し不合格（13件検出、要修正）。S-02（コンテナイメージスキャン）は2026-09-27に実施し不合格（Critical/High計18件検出、要修正）。S-06 はスキップ確定。その他は未実施

対応する手順書：[../procedures/security-test-procedure.md](../procedures/security-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) セキュリティ試験（S）

## 対象環境・構成
- S-01 / S-02：CI（tomario-app）と ECR、S-03：production の CloudFront、S-04 / S-05：各環境

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| S-01 | 依存パッケージの脆弱性スキャン | **不合格**（2026-09-26）。ローカルで`pip-audit -r requirements.txt`実行。4パッケージで既知脆弱性13件検出（重複エイリアス込みの生カウントは23件）：`flask==3.0.3`(CVE-2026-27205、修正版3.1.3)、`flask-cors==4.0.1`(CVE-2024-6221/6844/6866/6839の4件、修正版6.0.0)、`cryptography==42.0.8`(CVE-2026-26007/34073/69249/69248・CVE-2024-12797・GHSA 2件の計7件、同梱OpenSSLのバージョン起因が中心、修正版49.0.0)、`python-dotenv==1.0.1`(CVE-2026-28684、修正版1.2.2)。CI（`tomario-app`の`deploy.yml`）への組み込み自体は未実装（SEC-4未対応）のため、今回はローカル実行のみ | ⬜ 不合格 |
| S-02 | コンテナイメージの脆弱性スキャン | **不合格**（2026-09-27）。当初`describe-image-scan-findings --image-id imageTag=<tag>`で`ScanNotFoundException`となり原因調査。判明した原因：`docker buildx build --push`がECRに(1)実イメージ本体(2)attestationマニフェスト(3)両者をまとめるOCI image indexの3つを作成し、タグは(3)を指すが、スキャン結果は実際には(1)のdigestに紐づくため、タグ指定では見つからない仕様上の落とし穴だった。(1)のdigestを直接指定して再取得したところ、スキャン自体は正常に完了しており、**Critical 6件・High 12件・Medium 5件・Low 4件（計27件）**を検出。内訳は`perl`(6件)・`glibc`(2件、うちCritical含む)・`sqlite3`(2件)・`pcre2`(2件)・`zlib`(1件)で、全てベースイメージ（`python:3.12-slim`、Debian）のOSパッケージ由来（アプリコード自体の脆弱性ではない）。対応：Dockerfileに`apt-get upgrade`相当のステップを追加し、ベースイメージのOSパッケージを最新化する方針で修正予定 | ⬜ 不合格 |
| S-03 | TLS 設定の確認 | **スキップ**（2026-09-27決定）。`openssl s_client`でTLS1.0〜1.3の実ネゴシエーションを確認したところ、TLS1.0/1.1が有効なまま（cipherは`ECDHE-RSA-AES128-SHA`とSHA-1ベース）で、判定基準「TLS1.2以上のみ」を満たさない。原因は`modules/frontend/cloudfront.tf`の`viewer_certificate`が`cloudfront_default_certificate = true`（独自ドメイン無し）で、CloudFrontのデフォルト証明書使用時は`minimum_protocol_version`を絞れない仕様上の制約。**将来的に独自ドメインを取得する予定のため、その際にACM証明書へ切替えてTLS1.2以上に限定する対応とし、今回は見送り**（2026-09-12発見／2026-09-27判断確定） | ⬜ スキップ |
| S-04 | ネットワーク境界の構成確認 | (a) ALB 直アクセスが 403（SEC-7、X-Origin-Verify）は staging / production で確認済み。(c) ECS パブリック IP なしは確認済み。**(d) S3 Block Public Access は2026-09-27に確認完了**：`tomario-staging-logs-418295697340`・`tomario-staging-frontend`とも4項目（BlockPublicAcls/IgnorePublicAcls/BlockPublicPolicy/RestrictPublicBuckets）すべて`true`。(b) RDS:3306 の外部到達不可のみ未確認 | 🔺 一部（残りは(b)のみ） |
| S-05 | 脅威検知・監査証跡の実効性 | **合格**（2026-09-27、nonprod/production両方で実体確認完了）。<br>**nonprod/shared（account:418295697340）**：GuardDuty`Status:ENABLED`。CloudTrailのイベント記録は`RunTask`ではなく実際は`UpdateService`という名前で記録されると判明、2026-09-26のO-01/O-02/A-04試験時の`UpdateService`実行3件が実行者(`newport`)・時刻とも正確に記録、`TaskCreated`・`AutoScaling-RetrieveCurrentCapacity`等の関連イベントも追跡できた。S3保存：`s3://tomario-shared-cloudtrail-418295697340/`に本日分まで継続配信を確認。<br>**production（account:236782813946）**：GuardDuty`Status:ENABLED`。CloudTrailは`tomario-production-trail`（マルチリージョン）、2026-09-16のWAFアラーム実装デプロイ時の`UpdateService`（実行者`GitHubActions`）を発見。S3保存：`s3://tomario-production-logs-236782813946/`に本日分まで継続配信を確認 | ✅ |
| S-07 | WAF マネージドルールの有効性（production） | **合格**（2026-09-16、`test-summary.md`と同期・本ファイル反映漏れを2026-09-27訂正）。XSS（`<script>`タグPOST）・パストラバーサル（URLエンコード済みLFIペイロード）は`AWSManagedRulesCommonRuleSet`で`BLOCK`をWAFサンプリングログ（`get-sampled-requests`）で確認。SQLiは`AWSManagedRulesSQLiRuleSet`自体を導入していないため対象外（`tomario-app`はFlask-SQLAlchemyのORM経由で生SQL文字列組み立てが無く、アプリ側で既にパラメータ化されているため実害は低いと判断し、意図的に追加しない設計判断として整理済み）。PR #83（CloudFront Function移行）後はcurl応答が直接403になることも追加確認。判定はHTTPステータスコードでなくWAFサンプリングログで行う必要がある点に注意（下記前提事項参照） | ✅ |
| S-08 | WAF 誤検知（false positive）確認（production） | **スキップ**（`modules/waf`にCOUNT/BLOCK切替えの仕組みが未実装。production非公開期間で実ユーザーがいないためCOUNTモードで保護する必要性が薄く、BLOCKモードで直接有効化する運用に変更。一般公開後にCOUNT→ログ確認→BLOCK切替の仕組みを再検討する、`task-and-flow/remaining-task.md` #17参照） | ⬜ |
| S-09 | Security Hub の検出結果レビュー（production） | **合格**。Critical/High 13件を全件レビューし、各findingに対応方針（修正／スコープ外として許容／要確認）を記録。詳細は`task-and-flow/report/2026-09-12_security-hub-findings-review.md`（2026-09-12） | ✅ |
| S-10 | AWS Config ルールのコンプライアンス評価（production） | 未実施（`enable_config=true` 後、NON_COMPLIANT リソースをレビュー） | ⬜ |
| S-06 | IAM 最小権限の棚卸し（任意） | **スキップ**。元々「任意」項目であり、実施しない判断（2026-09-26決定） | ⬜ スキップ |

## 前提・未整備事項
- `pip-audit` は `tomario-app` 側への導入がまだ（`task-and-flow/remaining-task.md` SEC-4）
- **WAF ログの配信先（S3 / CloudWatch Logs / Firehose）が環境定義に未記載**。S-07 / S-08 / M-07 の前に決めて実装が必要
- WAF・Security Hub・AWS Config は production のみ・面接期間のみ有効化（`security-environment-design.md`）。S-07〜S-10 はその有効化後に実施
- bootstrap のインフラ用 IAM ポリシーに `wafv2` / `securityhub` / `config` 権限が含まれているか、導入時に確認する
- AWS 上の稼働環境に対して能動スキャンを行う場合は [AWS Customer Support Policy for Penetration Testing](https://aws.amazon.com/security/penetration-testing/) を確認し、禁止行為に該当しないことを確認してから実施する
- **CloudFront の `custom_error_response`（403/404→200、SPAルーティング対応）がWAFのBLOCK応答（403）まで200へマスキングする**（2026-09-12発見）。S-07/S-08 等でHTTPステータスコードによる合否判定は使えず、`aws wafv2 get-sampled-requests` 等のWAFログで`action`を確認する必要がある。恒久対応は `task-and-flow/remaining-task.md` #18（保留中）

## 派生した改善・課題
- `requirements.txt`のバージョン更新が必要：`flask`→3.1.3、`flask-cors`→6.0.0、`cryptography`→49.0.0、`python-dotenv`→1.2.2（S-01、2026-09-26発見）
- 依存関係スキャン（S-01）の CI 組み込みが未着手（`deploy.yml`への`pip-audit`ステップ追加、SEC-4）
- **`Dockerfile`にOSパッケージ更新ステップが無い**：ベースイメージ（`python:3.12-slim`）由来のCritical/High計18件が未パッチ（S-02、2026-09-27発見）。`apt-get update && apt-get upgrade`相当の追加で対応予定
- ECR イメージスキャン結果を定期レビューする運用が未整備。加えて、`docker buildx build --push`のマルチアーキ形式（image index）だと`describe-image-scan-findings`をタグ指定で呼んでも見つからない仕様上の落とし穴があると判明（S-02）。実イメージのdigestを指定する必要がある

## エビデンス
- `../evidence/security/` — 未取得（スキャンレポート・testssl.sh 出力・各確認コマンドの結果を実施時に格納。アカウント ID・エンドポイント等はマスク）
