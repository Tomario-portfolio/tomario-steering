# 非機能試験（監視）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の運用試験・監視（M-01〜M-08）
対応する結果報告書：[../results/monitoring-test-result.md](../results/monitoring-test-result.md)

## 試験実施目的

「CloudWatch アラームを作成した」「ログを出力している」ことと、「閾値超過時に実際に通知が届く」
「障害調査でログを追跡できる」ことは別問題である。監視が実運用で機能することを実測で確認する。

- SNS サブスクリプションが確認済みで、通知経路が開通していること
- アラームがメトリクスを取得できており、`INSUFFICIENT_DATA` で固まっていないこと
- 閾値を実際に超過させたときに `ALARM` へ遷移し、通知先メールまで届くこと
- CloudWatch Logs で特定のリクエスト・エラーを追跡できること

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging または production（prod 未公開期間）。手順内の名前は dev / staging を例に記載 |
| 起動状態 | cost-start 実行済みで ECS タスクが稼働中（アラーム評価に最低 10 分程度のメトリクス蓄積が必要） |
| 通知先 | SNS トピックにメールアドレスが登録済み |
| 注意 | M-03 で閾値を一時的に変更した場合、試験後に必ず元へ戻す。試験用アラームを作った場合は削除する |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| M-01 | SNS サブスクリプション確認 | dev / staging / production | `aws sns list-subscriptions-by-topic --topic-arn <alarm topic> --query "Subscriptions[*].{Protocol:Protocol,Endpoint:Endpoint,Arn:SubscriptionArn}"` | `SubscriptionArn` が `PendingConfirmation` ではなく実 ARN | 未確認だとアラーム発報しても通知が届かない |
| M-02 | アラームが OK 状態 | dev / staging / production | `aws cloudwatch describe-alarms --alarm-name-prefix "tomario-staging" --query "MetricAlarms[*].{Name:AlarmName,State:StateValue,Reason:StateReason}" --output table` | 全アラームの `StateValue` が `OK`（`INSUFFICIENT_DATA` でない） | 起動直後は評価期間 5 分 × 2 回で最大 10 分 `INSUFFICIENT_DATA` |
| M-03 | 閾値超過 → メール通知到達 | staging / production（未公開期間） | 詳細手順 1。対象アラームの閾値を一時的に下げる、または負荷をかけて実際に超過させる → `ALARM` 遷移を確認 → 通知先メール受信を確認 → 閾値を元に戻す | アラームが `ALARM` に遷移し、通知先メールでアラーム通知を受信する | production 初回動作確認（2026-09-03）では手動発火のみ実施済み。閾値超過による発報は本項目で実施 |
| M-04 | ログ追跡性（Logs Insights） | dev / staging / production | 詳細手順 2。CloudWatch Logs Insights で特定リクエスト ID・エラー文字列・ステータスコードで検索 | 該当するログ行を絞り込んで追跡できる | 「ログが出ている」と「調査に使える」は別 |
| M-05 | ECS アプリログの出力確認 | dev / staging / production | `aws logs describe-log-streams --log-group-name "/ecs/tomario-staging" --order-by LastEventTime --descending --max-items 3` → `aws logs get-log-events --log-group-name "/ecs/tomario-staging" --log-stream-name "<stream>" --limit 20 --query "events[*].message" --output text` | 直近の API リクエスト（`GET /health`・`POST /api/auth/login` 等）のログが出力されている | 保持日数は dev=7 / staging=30 / production=90 |
| M-06 | メトリクスダッシュボードの視認性 | staging | `tomario-staging-autoscaling` ダッシュボードを開き、負荷試験実施時間帯を含む範囲で CPU 使用率・タスク数の推移を確認 | 負荷と連動したグラフが視認できる | RunningTaskCount は Container Insights 未有効のため未表示（既知課題） |
| M-07 | WAF ログの配信確認 | production（未公開期間） | 詳細手順 3。WAF ログの配信先（S3 なら `aws s3 ls`、CloudWatch Logs なら `describe-log-streams` / Logs Insights）を確認し、サンプリングされたリクエストの `action`（ALLOW / BLOCK / COUNT）・`terminatingRuleId` が読めること | 配信先にログが存在し、S-07 で送った攻撃リクエストが `BLOCK` として記録されている | **配信先の設計が未記載＝先に S3 / CloudWatch Logs / Firehose のいずれか決めて実装が必要** |
| M-08 | WAF BlockedRequests アラートの発報 | production（未公開期間） | 詳細手順 4。`AWS/WAFV2` の `BlockedRequests` に CloudWatch アラームを設定 → S-07 の攻撃ペイロード送信 or 閾値を一時的に下げて `ALARM` に遷移させる → 通知先メール受信を確認 | `BlockedRequests` アラームが `ALARM` に遷移し、メール通知が届く | 攻撃検知時に気づける仕組みの確認。終了後に閾値を戻す |

## 詳細手順

### 手順 1（M-03）：閾値超過によるメール通知試験

```bash
# 対象アラームの現在の閾値を控える
aws cloudwatch describe-alarms --alarm-names tomario-staging-ecs-cpu \
  --query "MetricAlarms[0].{Threshold:Threshold,Period:Period,EvalPeriods:EvaluationPeriods}"

# 方法 A：閾値を一時的に下げて確実に超過させる（put-metric-alarm で同名上書き。他の属性も指定要）
#   → ALARM 遷移後、通知先メールを確認したら元の閾値で put-metric-alarm し直す
# 方法 B：k6 等で負荷をかけ、CPU を実際に閾値超まで持っていく（scalability 手順書と併用）

# 状態遷移の確認
aws cloudwatch describe-alarm-history --alarm-name tomario-staging-ecs-cpu \
  --history-item-type StateUpdate --max-records 5 \
  --query "AlarmHistoryItems[*].{Time:Timestamp,Summary:HistorySummary}" --output table
```

確認するもの：`OK → ALARM` の遷移が履歴に残ること、通知先メールに `ALARM:` 件名のメールが届くこと。
**終了後**：閾値を元に戻す。試験用に作成したアラームは `delete-alarms` で削除する。

### 手順 2（M-04）：ログ追跡性（CloudWatch Logs Insights）

```bash
aws logs start-query \
  --log-group-name "/ecs/tomario-staging" \
  --start-time $(date -v-1H +%s 2>/dev/null || date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
# 返る queryId を get-query-results に渡す
aws logs get-query-results --query-id <queryId>
```

確認するもの：エラー行から時刻・エンドポイント・スタックトレースまで辿れること。
特定リクエストの追跡は `filter @message like /<request id>/` 等で絞り込む。

### 手順 3（M-07）：WAF ログの配信確認

```bash
# 配信先が S3 の場合
aws s3 ls s3://<WAF ログバケット>/AWSLogs/ --recursive | tail
# 配信先が CloudWatch Logs の場合（ロググループ名は aws-waf-logs- で始まる必要がある）
aws logs describe-log-groups --log-group-name-prefix "aws-waf-logs-" \
  --query "logGroups[*].{name:logGroupName,retention:retentionInDays}"
aws logs start-query --log-group-name "aws-waf-logs-<name>" \
  --start-time $(date -v-1H +%s 2>/dev/null || date -d '1 hour ago' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, action, terminatingRuleId, httpRequest.uri | filter action="BLOCK" | limit 20'
aws logs get-query-results --query-id <queryId>
```

確認するもの：S-07 で送った攻撃リクエストが `action=BLOCK` で残っていること。ログが空なら配信設定（`wafv2 put-logging-configuration`）を見直す。

### 手順 4（M-08）：WAF BlockedRequests アラートの発報

```bash
# アラーム作成（例：5 分で BlockedRequests が 10 を超えたら ALARM）
aws cloudwatch put-metric-alarm \
  --alarm-name tomario-production-waf-blocked \
  --namespace AWS/WAFV2 --metric-name BlockedRequests \
  --dimensions Name=WebACL,Value=<web acl name> Name=Region,Value=CloudFront Name=Rule,Value=ALL \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions <SNS トピック ARN>

# S-07 の攻撃ペイロードを閾値を超える回数送る、または閾値を一時的に 0 にして発報させる
aws cloudwatch describe-alarm-history --alarm-name tomario-production-waf-blocked \
  --history-item-type StateUpdate --max-records 5 --output table
```

確認するもの：`OK → ALARM` 遷移と通知先メールの受信。**終了後**：閾値を運用値に戻す。

## 実施後の記録

- 結果を [../results/monitoring-test-result.md](../results/monitoring-test-result.md) に転記し、ステータスを「実施済み」に更新する
- M-03 のメール受信画面、M-02 のアラーム一覧を `../evidence/monitoring/` に格納する（メールアドレス等はマスク）
- 閾値を戻したこと・試験用アラームを削除したことをチェックする
