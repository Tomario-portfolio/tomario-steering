# リリースマネジメント

Tomario を「本番公開してよいか」判定してから、実際に切り替え、直後の稼働を見届けるまでの流れをまとめたディレクトリ。

`verification/`（非機能試験の計画・実施記録）が「品質を作り込む・実測する」フェーズだとすると、こちらは「作り込んだ品質を根拠に、公開の意思決定をする・安全に切り替える・切り替え後を見届ける」フェーズを扱う。実務のリリースマネジメント／変更管理（ITILでいうRelease & Change Management）に相当する。

Tomarioは実際にこのプロセスを本番運用として繰り返すわけではないため、**理解していることを示す成果物**として、tomarioの実際のインフラ構成（ECSローリングデプロイ＋デプロイサーキットブレーカー、digest指定のpromoteフロー、cost-start/cost-stop等）に即した内容で作成している。

## ドキュメント構成

| ファイル | 内容 |
|---|---|
| [release-criteria.md](release-criteria.md) | リリース判定基準。何が揃ったら公開してよいか、判定のタイミングと関係者 |
| [cutover-and-rollback.md](cutover-and-rollback.md) | カットオーバー手順（公開作業そのもの）と切り戻し手順 |
| [post-release-checklist.md](post-release-checklist.md) | 公開直後の稼働確認チェックリスト |
| [hypercare-and-handover.md](hypercare-and-handover.md) | ハイパーケア体制と、定常運用へ引き継ぐ際の項目 |

## 前提となる既存の仕組み

このドキュメント群は新しい仕組みを作るのではなく、既に`tomario-infra`・`tomario-app`に実装済みの以下の仕組みを「いつ・誰が・どう使うか」という運用の型にはめたもの。

- promoteフロー：`tomario-app`の`deploy.yml`の`promote-to-production`ジョブ。staging検証済みイメージをdigest指定でpull→再タグ→production ECRへpush（再ビルドしない）。GitHub Environment「prod」のRequired reviewersにより手動承認ゲートがかかる
- デプロイサーキットブレーカー：ECSのデプロイが異常な場合に自動ロールバックする仕組み（[A-01](../verification/non-functional-test/results/availability-test-result.md)で実測済み、MTTR約1分50秒）
- cost-start/cost-stop：`tomario-infra`の`cost-start.yml`/`cost-stop.yml`。環境の起動・停止を切り替える
- 非機能試験の合否・スキップ判断一式：[verification/non-functional-test/test-summary.md](../verification/non-functional-test/test-summary.md)（全43項目決着済み）
