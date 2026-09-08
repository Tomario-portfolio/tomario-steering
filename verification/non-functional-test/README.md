# 非機能試験

Tomario の非機能試験の計画と実績をまとめたディレクトリ。

prod 同等構成の staging 環境で、性能・可用性・信頼性・復旧・運用・セキュリティの各観点が設計どおりに
機能することを実測で確認し、その結果を prod へ引き継ぐ、という方針で実施している。

項番は `test-plan.md`（計画）・`procedures/`（実施手順書）・`results/`（結果報告書）で共通。

## ドキュメント構成

| ファイル | 内容 |
|---|---|
| [test-plan.md](test-plan.md) | 非機能試験計画書。試験方針・対象環境・確認項目一覧（実施前に定義） |
| [test-summary.md](test-summary.md) | 実施サマリ。全 44 項目の件数・実施済み／一部／未実施の一覧 |
| [procedures/](procedures/) | 試験種別ごとの実施手順書（目的・前提条件・試験項目一覧・詳細手順） |
| [results/](results/) | 試験種別ごとの結果報告書。手順書の各項番に対応する |
| [evidence/](evidence/) | スクリーンショット・ログ等のエビデンス |

`procedures/` `results/` `evidence/` は試験種別ごとに同じ名前で対応する（例：`scalability-test-procedure.md` ⇔ `scalability-test-result.md` ⇔ `evidence/scalability/`）。

### 試験種別と対応ドキュメント

| 試験種別 | 手順書 | 結果報告書 |
|---|---|---|
| スケーラビリティ（性能・負荷・Auto Scaling・目標スループット） | [scalability-test-procedure.md](procedures/scalability-test-procedure.md) | [scalability-test-result.md](results/scalability-test-result.md) |
| 可用性・信頼性（サーキットブレーカー・タスク強制停止・データ整合性・無停止デプロイ） | [availability-test-procedure.md](procedures/availability-test-procedure.md) | [availability-test-result.md](results/availability-test-result.md) |
| バックアップ・復旧（DR 訓練・PITR） | [backup-test-procedure.md](procedures/backup-test-procedure.md) | [backup-test-result.md](results/backup-test-result.md) |
| 監視（アラート発報・ログ追跡性） | [monitoring-test-procedure.md](procedures/monitoring-test-procedure.md) | [monitoring-test-result.md](results/monitoring-test-result.md) |
| 運用オペレーション（デプロイロールバック・インフラ再構築） | [operations-test-procedure.md](procedures/operations-test-procedure.md) | [operations-test-result.md](results/operations-test-result.md) |
| セキュリティ（脆弱性診断・ネットワーク境界・証跡） | [security-test-procedure.md](procedures/security-test-procedure.md) | [security-test-result.md](results/security-test-result.md) |

## 結果サマリ

| 試験種別 | 対象項番 | ステータス | 実施日 |
|---|---|---|---|
| スケーラビリティ | P-00・P-03〜P-05・P-08・P-09 | 🔺 一部実施（P-03 / P-04 / P-05 合格、P-00 / P-08 未実施、P-07 未達、P-09 は WAF 導入後） | 2026-07-20 |
| 可用性・信頼性 | A-01〜A-05 | 🔺 一部実施（A-01 / A-02 / A-03 合格、A-04 / A-05 未実施） | 2026-07-20 |
| バックアップ・復旧 | B-01〜B-06 | ✅ 実施済み・検証項目（B-01 / B-03 / B-04）全合格（B-06 任意未実施） | 2026-07-21 |
| 監視 | M-01〜M-08 | 🔺 一部実施（M-03 手動発火のみ。M-07 / M-08 は WAF 導入後、他は未実施） | 2026-09-03 |
| 運用オペレーション | O-01〜O-05 | 🔺 一部実施（O-02 を運用で実施、試験記録は未。O-04 / O-05 は WAF 導入後、O-01 未実施） | — |
| セキュリティ | S-01〜S-10 | 🔺 一部実施（S-04 / S-05 の一部のみ。S-07〜S-10 は WAF・Security Hub・Config 導入後、他は未実施） | — |

> **production 固有の追加検証**（WAF・Security Hub・AWS Config 導入に伴う）：P-09（WAF レイテンシ影響）／M-07・M-08（WAF ログ・アラート）／O-04・O-05（WAF の cost-stop/start 組み込み・緊急デタッチ）／S-07〜S-10（WAF ルール有効性・誤検知・Security Hub / Config レビュー）。詳細は各手順書と `test-plan.md` を参照。

## 用語について

- **機能試験** = 仕様どおりの動作をするか
- **非機能試験** = その動作を「どのくらいの品質で」できるか（性能・可用性・運用性・セキュリティ）。**セキュリティ試験・運用試験も非機能試験に含まれる**
- **疎通試験・設定確認**（ALB アクセスログ・VPC Flow Logs の S3 出力確認など）は品質を測るものではなく「そもそも繋がっているか」の確認のため、非機能試験ではなく**構築確認**として別途実施している（`tomario-workspace/reference/test/staging/layer4-logging.md`）

## 出典

本計画書・結果報告書は、`tomario-workspace/reference/test/staging/`（2026-07 時点で作成した試験項目・実施メモ）を、
実務標準の「試験計画書 → 結果報告書」の体裁に整理し直したもの。試験項目自体は実施（2026-07-20 / 21）より前に定義済み。
