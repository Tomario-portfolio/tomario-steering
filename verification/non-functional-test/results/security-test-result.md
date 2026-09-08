# 非機能試験（セキュリティ）結果報告書

## ステータス
一部実施済み。S-04 の一部（ALB 直アクセス 403）と S-05 の一部（GuardDuty / CloudTrail 有効化）は確認済み。その他は未実施

対応する手順書：[../procedures/security-test-procedure.md](../procedures/security-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) セキュリティ試験（S）

## 対象環境・構成
- S-01 / S-02：CI（tomario-app）と ECR、S-03：production の CloudFront、S-04 / S-05：各環境

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| S-01 | 依存パッケージの脆弱性スキャン | 未実施（`tomario-app` への `pip-audit` 導入自体が未対応＝ SEC-4） | ⬜ |
| S-02 | コンテナイメージの脆弱性スキャン | 未実施（ECR プッシュ時スキャンは有効だが結果レビュー運用が未整備。`describe-image-scan-findings` / Trivy で確認する） | ⬜ |
| S-03 | TLS 設定の確認 | 未実施（testssl.sh / SSL Labs で CloudFront を診断予定） | ⬜ |
| S-04 | ネットワーク境界の構成確認 | (a) ALB 直アクセスが 403（SEC-7、X-Origin-Verify）は staging / production で確認済み。(b) RDS:3306 の外部到達不可・(c) ECS パブリック IP なし・(d) S3 Block Public Access はまとめての再確認が未 | 🔺 一部 |
| S-05 | 脅威検知・監査証跡の実効性 | GuardDuty・CloudTrail が有効化されていることは確認済み（production 初回動作確認）。CloudTrail のイベント記録・S3 保存の実体確認は未 | 🔺 一部 |
| S-07 | WAF マネージドルールの有効性（production） | 未実施（WAF 導入後、SQLi / XSS / KnownBadInputs 相当のペイロードで 403 ブロックを確認） | ⬜ |
| S-08 | WAF 誤検知（false positive）確認（production） | 未実施（WAF 導入後、COUNT モードで正常業務を流しログ確認 → 除外設定 → BLOCK へ切替） | ⬜ |
| S-09 | Security Hub の検出結果レビュー（production） | 未実施（`enable_security_hub=true` 後、Critical / High findings をレビュー） | ⬜ |
| S-10 | AWS Config ルールのコンプライアンス評価（production） | 未実施（`enable_config=true` 後、NON_COMPLIANT リソースをレビュー） | ⬜ |
| S-06 | IAM 最小権限の棚卸し（任意） | 未実施（IAM Access Analyzer の未使用アクセス分析。優先度低） | ⬜ |

## 前提・未整備事項
- `pip-audit` は `tomario-app` 側への導入がまだ（`task-and-flow/remaining-task.md` SEC-4）
- **WAF ログの配信先（S3 / CloudWatch Logs / Firehose）が環境定義に未記載**。S-07 / S-08 / M-07 の前に決めて実装が必要
- WAF・Security Hub・AWS Config は production のみ・面接期間のみ有効化（`security-environment-design.md`）。S-07〜S-10 はその有効化後に実施
- bootstrap のインフラ用 IAM ポリシーに `wafv2` / `securityhub` / `config` 権限が含まれているか、導入時に確認する
- AWS 上の稼働環境に対して能動スキャンを行う場合は [AWS Customer Support Policy for Penetration Testing](https://aws.amazon.com/security/penetration-testing/) を確認し、禁止行為に該当しないことを確認してから実施する

## 派生した改善・課題
- 依存関係スキャン（S-01）の CI 組み込みが未着手
- ECR イメージスキャン結果を定期レビューする運用が未整備

## エビデンス
- `../evidence/security/` — 未取得（スキャンレポート・testssl.sh 出力・各確認コマンドの結果を実施時に格納。アカウント ID・エンドポイント等はマスク）
