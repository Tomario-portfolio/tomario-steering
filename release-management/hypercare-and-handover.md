# ハイパーケア・運用引き継ぎ体制

最終更新：2026-09-30

[post-release-checklist.md](post-release-checklist.md)完了後、リリース直後の集中監視期間（ハイパーケア）から、定常運用（[operations](../verification/non-functional-test/procedures/operations-test-procedure.md)で確認済みの通常運用体制）へ引き継ぐまでの体制。

## 1. ハイパーケア期間の監視体制

- **期間の目安**：カットオーバー完了から24〜72時間（実務ではリリース規模・過去の障害履歴に応じて調整）
- **監視頻度**：通常運用時より高頻度（例：数時間おきにダッシュボード・アラーム状態を確認）
- **監視対象**：[post-release-checklist.md](post-release-checklist.md)の1〜4と同じ項目を継続的に確認
- **チャネル**：CloudWatch AlarmsからSNS経由でメール通知（[M-01〜M-03](../verification/non-functional-test/results/monitoring-test-result.md)で疎通確認済み）

## 2. エスカレーションフロー

| 検知内容 | 一次対応 | 判断基準 |
|---|---|---|
| CloudWatchアラームが1件ALARM | 該当メトリクスを確認し、一時的なスパイクか継続的な悪化かを判断 | 5分以内に自然回復すれば静観、継続する場合は次の段階へ |
| 5xxエラーの急増・サービス断 | ログ確認 → 原因が直近の変更に起因すると判断できれば即座に[cutover-and-rollback.md](cutover-and-rollback.md)の切り戻しを実施 | 影響がユーザー体験に及ぶ場合は判断を待たず切り戻しを優先する |
| WAFの誤検知でユーザーがブロックされる | [O-03（WAF緊急デタッチ手順）](../verification/non-functional-test/results/operations-test-result.md)を実施 | 正規リクエストのブロック率が明確に異常な場合 |
| RDSの異常（接続断・CPU高負荷） | RDSメトリクス確認、必要に応じて[B-03（PITR）](../verification/non-functional-test/results/backup-test-result.md)による復旧を検討 | データ破損・接続不能が継続する場合 |

本ポートフォリオでは判定者・実施者・監視担当を1人で兼務するため、実際のエスカレーション先（他チーム・オンコール担当）は存在しない。実務では、この表の「一次対応」欄が「誰が」「どのSlackチャネル/オンコールツールで」動くかまで具体化される。

## 3. 定常運用への引き継ぎ項目

ハイパーケア期間が過ぎ、異常がないことを確認できたら、以下を確認して通常運用体制（監視頻度を通常に戻す）へ移行する。

- [ ] ハイパーケア期間中に発生したインシデント・対応内容を記録する（`tomario-workspace/history/activity_log/`に記載）
- [ ] 今回のリリースで新たに判明した問題・改善点があれば`tomario-workspace/task-and-flow/remaining-task.md`に追加する
- [ ] ランブック（障害対応手順）に更新が必要な変更がなかったか確認する。既存の関連ドキュメント：
  - デプロイロールバック手順：[operations-test-procedure.md](../verification/non-functional-test/procedures/operations-test-procedure.md)
  - WAF緊急デタッチ手順：同上
  - ログ多行スタックトレースの手動反映手順：`tomario-workspace/task-and-flow/m04-multiline-pattern-manual-reflection.md`
- [ ] 監視ダッシュボード・アラーム閾値が今回のリリース内容に対して引き続き適切か確認する
- [ ] リリース判定基準（[release-criteria.md](release-criteria.md)）自体に見直しが必要な学びがなかったか確認する
