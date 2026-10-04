# 004. RDS Performance Insightsの導入見送り

## ステータス
見送り（インスタンスクラスの制約により技術的に実現不可、再検討の予定なし）

## コンテキスト
負荷テスト（P-08等）のボトルネック分析のため、RDSのクエリ単位の性能分析ができるPerformance Insights（7日保持は無料枠）の導入を検討した。2026-09-29に`modules/database`（全環境共通モジュール）へ`performance_insights_enabled=true`を追加し、`terraform plan`は正常に通過したため適用を試みた。

## 決定
Performance Insightsは導入しない。該当のTerraform変更は全てリバートした。

## 選定理由
`terraform apply`の段階で`InvalidParameterCombination: Performance Insights not supported for this configuration`エラーが発生。`aws rds describe-orderable-db-instance-options`で確認したところ、`db.t3.micro`・`db.t3.small`・`db.t4g.micro`（MySQL 8.4.9）ではPerformance Insights自体がサポートされておらず、`db.t4g.medium`以上のインスタンスクラスが必要と判明した。

- **インスタンスクラスを恒久的に引き上げる**：Performance Insightsは使えるようになるが、垂直スケールをしない（水平スケールのみで対応する）というコスト最適化の基本方針に反し、全環境のRDSコストが恒常的に増加する
- **導入を見送る**：性能分析の手段としては失うが、既存の非機能試験（P-08等）でCloudWatchメトリクス（CPU使用率等）による原因特定で十分対応できている

## 利点
- インスタンスクラスを変更せず、コスト最適化の方針を維持できる
- `terraform plan`では検知できない「apply時にしか分からないAWS側の制約」として、設計レビューの教訓にできる

## 欠点
- クエリ単位の詳細な性能分析（スロークエリの特定等）ができない
- 将来、本格的な性能チューニングが必要になった場合は、改めてインスタンスクラスの引き上げを含めて再検討する必要がある

## 関連情報
- 設計の詳細：[database-high-level-spec.md](../../../../tomario-docs/basic-design/database-high-level-spec.md)・[monitoring-high-level-spec.md](../../../../tomario-docs/basic-design/monitoring-high-level-spec.md)
- `aws rds describe-orderable-db-instance-options --engine mysql --engine-version 8.4.9 --db-instance-class db.t3.micro`で`SupportsPerformanceInsights: false`を確認（t3.small/t4g.microも同様、t4g.medium以上で`true`）
