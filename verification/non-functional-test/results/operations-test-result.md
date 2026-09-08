# 非機能試験（運用：ロールバック・インフラ再構築）結果報告書

## ステータス
一部実施済み。O-02（cost-stop / cost-start）は日常運用で繰り返し実施しているが、試験としての記録は未。O-01 は未実施

対応する手順書：[../procedures/operations-test-procedure.md](../procedures/operations-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 運用試験（オペレーション）（O）

## 対象環境・構成
- staging（`tomario-staging-*`）。O-01 は production の promote フローにも読み替え可能

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| O-01 | デプロイロールバック手順の実演 | 未実施（`update-service --task-definition <family>:<前revision> --force-new-deployment` → `wait services-stable` → 疎通確認、所要時間を記録する） | ⬜ |
| O-02 | cost-stop / cost-start によるインフラ再構築確認 | cost-stop / cost-start は staging・production で繰り返し実施しており、cost-start 後に `terraform plan` 差分ゼロ・CloudFront 経由の疎通を都度確認している。**「試験」として所要時間・CloudFront 再作成の有無を記録した実績は未** | 🔺 一部 |
| O-04 | WAF Web ACL の cost-stop/start 組み込み（production） | 未実施（WAF 導入後。cost-start で作成＋関連付け、cost-stop で解除＋削除が回ること、`terraform plan` 差分ゼロを確認） | ⬜ |
| O-05 | WAF 緊急デタッチ手順（production） | 未実施（WAF 導入後。`disassociate-web-acl` でアクセス復旧できることを実演し、ランブックとして所要時間を記録） | ⬜ |
| O-03 | ロールバック後の復帰（後始末） | 未実施（O-01 実施後に最新リビジョンへ戻す／様子見の判断を記録する） | ⬜ 後始末 |

## 補足
- O-02：cost-stop の destroy 連鎖で CloudFront も再作成対象になり、cost-start のたびに CloudFront の新規作成に 20〜30 分かかることがある（`tomario-workspace/reference/test/staging/output.md` に記録あり）。試験実施時はこの所要時間を計測して記録する
- O-01：production では staging で検証済みイメージの digest promote を実装済みのため、その逆順（1 つ前の digest への re-tag → タスク定義更新）が切り戻し手順になる

## 派生した改善・課題
- ロールバック手順（O-01）の実演・所要時間記録が未実施
- cost-stop の CloudFront 道連れ destroy は既知の非効率。cost-start の所要時間を押し上げている（別途対応検討）

## エビデンス
- `../evidence/operations/` — 未取得（`describe-services` の状態遷移、`terraform plan` の差分ゼロ出力、疎通確認結果を実施時に格納）
