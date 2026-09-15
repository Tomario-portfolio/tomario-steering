# 非機能試験（監視）結果報告書

## ステータス
一部実施済み。M-03 のうち手動発火のみ確認済み（2026-09-03、production 初回動作確認）。閾値超過による発報・その他項目は未実施

対応する手順書：[../procedures/monitoring-test-procedure.md](../procedures/monitoring-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 運用試験（監視）（M）

## 対象環境・構成
- staging または production（prod 未公開期間）
- CloudWatch アラーム（ECS / ALB / RDS）→ SNS → メール通知
- ECS アプリログ → CloudWatch Logs（保持 dev=7 / staging=30 / production=90 日）

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| M-01 | SNS サブスクリプション確認 | 未実施（`list-subscriptions-by-topic` で `PendingConfirmation` でないことを確認する） | ⬜ |
| M-02 | アラームが OK 状態 | 未実施（`describe-alarms` で全アラームが `OK`＝`INSUFFICIENT_DATA` で固まっていないことを確認する） | ⬜ |
| M-03 | 閾値超過 → メール通知到達 | production 初回動作確認で「**手動でアラーム状態を作る → SNS → メール通知**」は 1 回確認済み。メトリクス閾値を実際に超過させて `ALARM` 遷移させる発報試験は未実施 | 🔺 一部 |
| M-04 | ログ追跡性（Logs Insights） | 未実施（特定リクエスト ID・エラー文字列で絞り込み追跡できることを確認する） | ⬜ |
| M-05 | ECS アプリログの出力確認 | 一部確認済み（staging layer4 で `/ecs/tomario-staging` の保持 30 日は確認）。ログ内容から直近リクエストを追う確認は未 | 🔺 一部 |
| M-06 | メトリクスダッシュボードの視認性 | CPU 使用率ウィジェットは表示可。**RunningTaskCount は Container Insights 未有効のため未表示**（`scalability-test-result.md` P-07 と同一課題） | 🔺 一部 |
| M-07 | WAF ログの配信確認（production） | **合格**。配信先は`modules/waf/logging.tf`でCloudWatch Logs（`aws-waf-logs-tomario-production-{cloudfront,alb}`）に実装済みだった（ドキュメント未反映だっただけ）。実際にログイベントが記録され、`action`（ALLOW/BLOCK）・`httpRequest`（uri・args・clientIp等）が読めることを確認（2026-09-12） | ✅ |
| M-08 | WAF BlockedRequests アラートの発報（production） | 未実施（WAF 導入後、`BlockedRequests` にアラームを設定し攻撃検知で発報を確認） | ⬜ |

## 補足
- M-03 実施時は「閾値を戻し忘れない」「試験用アラームを消し忘れない」ことに注意
- production 初回動作確認での手動発火は「SNS 経路（トピック → メール購読）が生きている」ことの確認にはなっているが、「アラームの閾値判定が正しく働く」ことの確認にはなっていない

## 派生した改善・課題
- 閾値超過による実発報試験（M-03 の本来の形）が未実施
- RunningTaskCount 未表示（Container Insights 未有効）— 導入を検討

## エビデンス
- `../evidence/monitoring/` — 未取得（M-03 のメール受信画面、M-02 のアラーム一覧、M-04 の Logs Insights 結果を実施時に格納）
