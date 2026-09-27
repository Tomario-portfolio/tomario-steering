# 非機能試験（監視）結果報告書

## ステータス
一部実施済み。M-01・M-02（staging、2026-09-26）が合格に追加。M-03 のうち手動発火のみ確認済み（2026-09-03、production 初回動作確認）。M-04（staging、2026-09-26）は一部合格。閾値超過による発報・その他項目は未実施

対応する手順書：[../procedures/monitoring-test-procedure.md](../procedures/monitoring-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 運用試験（監視）（M）

## 対象環境・構成
- staging または production（prod 未公開期間）
- CloudWatch アラーム（ECS / ALB / RDS）→ SNS → メール通知
- ECS アプリログ → CloudWatch Logs（保持 dev=7 / staging=30 / production=90 日）

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| M-01 | SNS サブスクリプション確認（staging） | **合格**。`list-subscriptions-by-topic` で `masashi00aws@gmail.com` の `SubscriptionArn` が `PendingConfirmation` ではなく実 ARN であることを確認（2026-09-26） | ✅ |
| M-02 | アラームが OK 状態（staging） | **合格**。`tomario-staging-{alb-unhealthy-host,ecs-cpu,rds-cpu}` の3件とも `StateValue: OK`（`INSUFFICIENT_DATA` なし）（2026-09-26） | ✅ |
| M-03 | 閾値超過 → メール通知到達 | production 初回動作確認で「**手動でアラーム状態を作る → SNS → メール通知**」は 1 回確認済み。メトリクス閾値を実際に超過させて `ALARM` 遷移させる発報試験は未実施 | 🔺 一部 |
| M-04 | ログ追跡性（Logs Insights、staging） | **一部合格**（2026-09-26）。実測手順・結果は下記「M-04 詳細」参照。時刻・エンドポイント・ステータスコードは1行のログで辿れる。一方リクエストID・パラメータ・スタックトレースは無く、個別リクエストの追跡や例外発生時の原因追跡はできない | 🔺 一部 |
| M-05 | ECS アプリログの出力確認 | **合格**（2026-09-27）。`describe-log-streams`→`get-log-events`で直近ログストリームから`GET /health`等のリクエストログを実際に取得・追跡できることを確認。加えてM-04調査時に`GET /api/hotels`等の実APIリクエストもログに残っていることを確認済み。追加実装不要 | ✅ |
| M-06 | メトリクスダッシュボードの視認性 | CPU 使用率ウィジェットは表示可。**RunningTaskCount は Container Insights 未有効のため未表示**（`performance-test-result.md` P-07 と同一課題）。**2026-09-27、Container Insights導入をコスト試算の上で決定**（月10時間程度の稼働なら月10〜15セント程度、`remaining-task.md` #1・`verification/env/remain.md`参照）。実装・再確認待ち | 🔺 一部（対応方針確定） |
| M-07 | WAF ログの配信確認（production） | **合格**。配信先は`modules/waf/logging.tf`でCloudWatch Logs（`aws-waf-logs-tomario-production-{cloudfront,alb}`）に実装済みだった（ドキュメント未反映だっただけ）。実際にログイベントが記録され、`action`（ALLOW/BLOCK）・`httpRequest`（uri・args・clientIp等）が読めることを確認（2026-09-12） | ✅ |
| M-08 | WAF BlockedRequests アラートの発報（production） | 未実施（WAF 導入後、`BlockedRequests` にアラームを設定し攻撃検知で発報を確認） | ⬜ |

## M-04 詳細（staging、2026-09-26）

**手順（`monitoring-test-procedure.md`手順2の修正版）**：
1. 生ログの形式を確認 → `/ecs/tomario-staging`に出ていたのはWerkzeug（Flask開発用サーバー）標準のアクセスログ形式のみ（`<IP> - - [日時] "メソッド パス HTTP/1.1" ステータス -`）。`tomario-app`の`app.py`にはログ／例外処理・request-id付与の実装が無く、独自のエラーログは出ていないことが判明
2. わざとエラーを起こす：`curl "https://d14h67xxnvmsdi.cloudfront.net/api/this-does-not-exist"` → クライアントから見えた応答は`200`
3. ログで実際のバックエンド応答を確認：同じリクエストのログ行は`"GET /api/this-does-not-exist HTTP/1.1" 404`で、バックエンド（Flask）は正しく404を処理していた

**重要な発見**：CloudFrontの`custom_error_response`（SPAルーティング対応、`remaining-task.md` #18で既知）により、クライアントに返る応答は`200`にマスキングされる。**curl等のHTTPステータスコードでは合否判定できず、CloudWatch Logsで実際のオリジン応答を確認する必要がある**（S-07で判明した制約と同じ構造の問題が、監視・ログ調査の文脈でも再確認された）

**確認できたこと**：
- 時刻・メソッド・パス・ステータスコードは1行のログから辿れる（○）
- 特定リクエストIDでの絞り込み（`filter @message like /<request id>/`）：request-idの仕組みが無いため不可（✕）
- 例外発生時のスタックトレース追跡：今回は404のみで例外（500系）は未発生のため未確認。ECS側のログ設定（`modules/backend/ecs.tf`）に`awslogs-multiline-pattern`が未設定なため、仮に複数行のスタックトレースが出力されても1行＝1イベントに分断される見込み

## 補足
- M-03 実施時は「閾値を戻し忘れない」「試験用アラームを消し忘れない」ことに注意
- production 初回動作確認での手動発火は「SNS 経路（トピック → メール購読）が生きている」ことの確認にはなっているが、「アラームの閾値判定が正しく働く」ことの確認にはなっていない

## 派生した改善・課題
- 閾値超過による実発報試験（M-03 の本来の形）が未実施
- RunningTaskCount 未表示（Container Insights 未有効）— 導入を検討
- **`tomario-app`が本番相当環境でFlask開発用サーバー（`app.run()`）を直接使っている**（gunicorn等の本番向けWSGIサーバー未導入）。request-id付与・構造化ログ・`@app.errorhandler`によるエラーログも無く、M-04で「調査に使えるログか」の観点では不十分と判明。対応候補：`Dockerfile`のCMDをgunicornに変更（`--access-logfile=-`だけでもログ改善効果あり）、将来的に構造化ログ・request-id対応（2026-09-26発見、`tomario-workspace/task-and-flow/remaining-task.md`へ起票予定）
- CloudFrontの`custom_error_response`によるステータスコードマスキング（`remaining-task.md` #18）が、外形監視・障害調査でも影響することを再確認。今後の運用試験・監視設計は「クライアント視点のステータスコード」ではなく「オリジン側のログ」を正とする前提で行う必要がある

## エビデンス
- `../evidence/monitoring/` — 未取得（M-03 のメール受信画面、M-02 のアラーム一覧、M-04 の Logs Insights 結果を実施時に格納）
