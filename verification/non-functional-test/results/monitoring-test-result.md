# 非機能試験（監視）結果報告書

## ステータス
全項目合格（M-01〜M-08）

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
| M-03 | 閾値超過 → メール通知到達（staging） | **合格**（2026-09-29）。`tomario-staging-ecs-cpu`の閾値を80%→0.01%に一時変更し、5分×2回連続の実データで`OK→ALARM`遷移を確認（`StateReason`に実測値2件記載）、メール通知の受信も確認。試験後、閾値を80%へ復元済み | ✅ |
| M-04 | ログ追跡性（Logs Insights、staging） | **合格**（2026-09-29、`awslogs-multiline-pattern`反映後に再検証）。実測手順・結果は下記「M-04 詳細」参照。時刻・エンドポイント・ステータスコード・スタックトレース全体まで1つの`@message`で辿れることを確認。request-id自体は未実装のため、エラーメッセージに区別できる情報が無いケースでの厳密な個別リクエスト追跡は依然できない（既知の制約として記録） | ✅ |
| M-05 | ECS アプリログの出力確認 | **合格**（2026-09-27）。`describe-log-streams`→`get-log-events`で直近ログストリームから`GET /health`等のリクエストログを実際に取得・追跡できることを確認。加えてM-04調査時に`GET /api/hotels`等の実APIリクエストもログに残っていることを確認済み。追加実装不要 | ✅ |
| M-06 | メトリクスダッシュボードの視認性 | **合格**（2026-09-29）。Container Insights導入（`tomario-infra` PR #89、2026-09-28マージ）後、`aws ecs describe-clusters`で`containerInsights: enabled`を確認、`ECS/ContainerInsights`名前空間の`RunningTaskCount`メトリクスが実際に配信されている（直近は2.0＝タスク2台）ことを確認。ダッシュボードのRunningTaskCountウィジェットも表示される見込み | ✅ |
| M-07 | WAF ログの配信確認（production） | **合格**。配信先は`modules/waf/logging.tf`でCloudWatch Logs（`aws-waf-logs-tomario-production-{cloudfront,alb}`）に実装済みだった（ドキュメント未反映だっただけ）。実際にログイベントが記録され、`action`（ALLOW/BLOCK）・`httpRequest`（uri・args・clientIp等）が読めることを確認（2026-09-12） | ✅ |
| M-08 | WAF BlockedRequests アラートの発報（production） | **合格**（2026-09-16）。攻撃ペイロード30リクエストで`OK→ALARM`遷移・SNSメール通知の受信まで実機確認。実装過程でRegionディメンション誤り・アラームのリージョン誤り・SNSトピックのリージョン不一致の3件のバグを発見・修正（`tomario-infra` PR #84〜#88） | ✅ |

## M-04 再検証（staging、2026-09-29）

`tomario-infra` PR #90（`awslogs-multiline-pattern`追加）マージ後、稼働中のタスク定義へ手動反映（`tomario-workspace/task-and-flow/m04-multiline-pattern-manual-reflection.md`手順、revision:11→13）。

`/api/hotels?check_in=not-a-date`と`/api/hotels?check_in=also-bad`をほぼ同時（1ms差）に実行し、Logs Insightsで確認：
- 各トレースバック（`ERROR in app:`〜`ValueError:`まで5〜6行）が**1つの`@message`にまとまっている**ことを確認（修正前は1行＝1イベントに分断されていた）
- 2件のエラーは、たまたま例外メッセージに実際の不正値（`'also-bad'`/`'not-a-date'`）が含まれていたため区別できた。ただし例外メッセージに区別情報が無いケース（DB接続エラー等）では、request-id未実装のため依然として個別リクエストの特定はできない

## M-04 詳細（staging、2026-09-26、初回調査）

**手順（`monitoring-test-procedure.md`手順2の修正版）**：
1. 生ログの形式を確認 → `/ecs/tomario-staging`に出ていたのはWerkzeug（Flask開発用サーバー）標準のアクセスログ形式のみ（`<IP> - - [日時] "メソッド パス HTTP/1.1" ステータス -`）。`tomario-app`の`app.py`にはログ／例外処理・request-id付与の実装が無く、独自のエラーログは出ていないことが判明
2. わざとエラーを起こす：`curl "https://d14h67xxnvmsdi.cloudfront.net/api/this-does-not-exist"` → クライアントから見えた応答は`200`
3. ログで実際のバックエンド応答を確認：同じリクエストのログ行は`"GET /api/this-does-not-exist HTTP/1.1" 404`で、バックエンド（Flask）は正しく404を処理していた

**重要な発見**：CloudFrontの`custom_error_response`（SPAルーティング対応）により、クライアントに返る応答は`200`にマスキングされる（403→200は`tomario-infra` PR #83で解消済み。404→200はSPA用に残っている）。**curl等のHTTPステータスコードでは合否判定できず、CloudWatch Logsで実際のオリジン応答を確認する必要がある**（S-07で判明した制約と同じ構造の問題が、監視・ログ調査の文脈でも再確認された）

**確認できたこと**：
- 時刻・メソッド・パス・ステータスコードは1行のログから辿れる（○）
- 特定リクエストIDでの絞り込み（`filter @message like /<request id>/`）：request-idの仕組みが無いため不可（✕）
- 例外発生時のスタックトレース追跡：今回は404のみで例外（500系）は未発生のため未確認。ECS側のログ設定（`modules/backend/ecs.tf`）に`awslogs-multiline-pattern`が未設定なため、仮に複数行のスタックトレースが出力されても1行＝1イベントに分断される見込み

## 補足
- M-03 実施時は「閾値を戻し忘れない」「試験用アラームを消し忘れない」ことに注意
- production 初回動作確認での手動発火は「SNS 経路（トピック → メール購読）が生きている」ことの確認にはなっているが、「アラームの閾値判定が正しく働く」ことの確認にはなっていない

## 派生した改善・課題
- ~~RunningTaskCount 未表示~~：Container Insights導入で解消済み（2026-09-29、`tomario-infra` PR #89）
- ~~`tomario-app`がFlask開発用サーバーのまま~~：gunicornへ移行済み（2026-09-28、`tomario-app` PR #13・#14）。あわせて`awslogs-multiline-pattern`も反映済み（PR #90）
- **request-id付与・構造化ログ・`@app.errorhandler`は依然未実装**：例外メッセージに区別情報が無いケースでは個別リクエストの追跡ができない、という制約が2026-09-29の再検証で確認された。対応候補は変わらず（`@app.before_request`でrequest-id付与、`@app.errorhandler(Exception)`でrequest-id付きエラーログ出力）
- CloudFrontの`custom_error_response`によるステータスコードマスキング（403はPR #83で解消済み、404→200は残存）が、外形監視・障害調査でも影響することを再確認。今後の運用試験・監視設計は「クライアント視点のステータスコード」ではなく「オリジン側のログ」を正とする前提で行う必要がある

## エビデンス
- `../evidence/monitoring/` — 未取得（M-03 のメール受信画面、M-02 のアラーム一覧、M-04 の Logs Insights 結果を実施時に格納）
