# 非機能試験（可用性）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の可用性・信頼性試験（A-01〜A-04）
対応する結果報告書：[../results/availability-test-result.md](../results/availability-test-result.md)

## 試験実施目的

`deployment_circuit_breaker { enable = true, rollback = true }` を設定していることと、
実際に障害時に自動復旧が機能することは別問題である。壊れたデプロイやタスク停止を意図的に発生させ、
自動復旧の仕組みが本当に動くか、その際にデータ不整合が生じないかを実測で確認する。

- 不正なデプロイをデプロイサーキットブレーカーが検知し、直前の正常リビジョンへ自動ロールバックすること
- 稼働中タスクを強制停止しても、代替タスクが自動起動して冗長数を回復すること
- タスク停止がリクエスト処理中に発生しても、DB に中途半端なレコードが残らないこと
- 正常なローリングデプロイの最中もリクエストが継続処理される（無停止デプロイ）こと

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging（`tomario-staging-cluster` / `tomario-staging-service`、desired_count=2）。dev/staging 共通設定だが、2 台構成の staging の方が「1 台落ちても残り 1 台で捌ける」ことを確認しやすい |
| 起動状態 | cost-start 実行済み。CloudFront ドメイン名を控えておく |
| テストデータ | A-03 で使うため users 2,000 件投入済み（`loadtest_user_0@example.com` 等） |
| コンテナ接続 | A-03 の DB 確認で ECS Exec を使う（`ssmmessages` VPC エンドポイントが存在すること） |
| 注意 | A-01 で登録した壊れたタスク定義リビジョンは自動削除されない。試験後に `deregister-task-definition` で片付ける |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| A-01 | デプロイサーキットブレーカーによる自動ロールバック | staging | 1. 現在のタスク定義を取得し、コンテナの起動コマンドを存在しないコマンドに書き換える。それを新しいリビジョン（壊れたタスク定義）として登録する<br>2. ECSサービスを壊れたリビジョンでデプロイし、デプロイ開始時刻を記録する<br>3. デプロイの状態を10秒間隔で監視する。デプロイ失敗を検知した時刻と、直前の正常なリビジョンへのロールバックが完了した時刻を記録する<br>4. ロールバックの完了後、稼働中のタスク数が2台に戻っていることを確認する。あわせて、サービスのイベントログにロールバックの記録が残っていることを確認する<br>5. 失敗検知からロールバック完了までの時間を算出する<br>（コマンドは詳細手順1を参照） | `rolloutState` が `IN_PROGRESS` → `FAILED`（reason: circuit breaker threshold reached）→ 直前の正常リビジョンへ自動ロールバック。ロールバック中も `running` が 0 にならない。失敗検知からロールバック完了まで 5 分未満 |  |
| A-02 | タスク強制停止からの自動復旧 | staging | 1. 稼働中のタスクを1つ選んで強制停止し、停止した時刻を記録する<br>2. サービスのタスク数（必要数・稼働中・起動中）を5秒間隔で監視し、稼働中のタスク数が2台に戻った時刻を記録する<br>3. 2台に戻ったあと、ALBのターゲットグループで、2台ともヘルスチェックが正常（healthy）であることを確認する<br>4. 停止から2台に戻るまでの時間を、復旧時間（MTTR）として算出する<br>（コマンドは詳細手順2を参照） | 停止直後 `running:1` → `pending:1` → 1〜2 分で `running:2`。ALB ターゲット 2 台とも `healthy`。MTTR < 5 分 | AZ 障害・ホスト障害の簡易シミュレーション |
| A-03 | 処理中リクエストへの影響・データ整合性 | staging | 1. ターミナルで、試験用ユーザーでログインする<br>2. ターミナルで、予約作成APIへ0.3秒間隔で100回リクエストを送る<br>3. 送信の最中に、別のターミナルで稼働中のタスクを1つ強制停止する<br>4. 送信が終わったら、ステータスコードごとの件数を数える。5xxが返ったリクエストは、レスポンスの内容を確認する<br>5. ECS Execでコンテナに入り、予約テーブルに不完全なレコード（金額が空や0など）が残っていないことを確認する<br>（コマンドは詳細手順3を参照） | エラー（5xx）は数件まで許容（`deregistration_delay=30` のため瞬間的なエラーは想定内）。**DB に `total_price` が NULL/0 等の不完全なレコードが残らない**。タスク復旧後は `201` が再び返る |  |
| A-04 | 正常なローリングデプロイ中の無停止性 | staging | 1. ターミナルで、ヘルスチェック用のURL（`/health`）へ0.3秒間隔で400回リクエストを送る<br>2. 送信の最中に、別のターミナルでタスク定義を変えずに再デプロイを実行し、デプロイ開始時刻を記録する<br>3. 別のターミナルで、サービスのデプロイ数とタスク数を5秒間隔で監視する。稼働中のタスク数が0台にならないことと、デプロイが1本に収束することを確認し、収束した時刻を記録する<br>4. 送信が終わったら、ステータスコードごとの件数を数える<br>（コマンドは詳細手順4を参照） | デプロイ中も 5xx がほぼ発生しない（数件以内）。`running` が 0 にならず、`deployments` が 1 本（PRIMARY のみ）に収束する | 異常系（A-01）だけでなく正常デプロイ時のリクエスト継続性を確認 |

## 詳細手順

### 手順 1（A-01）：デプロイサーキットブレーカー

```bash
# 現行タスク定義を取得し、起動直後にクラッシュする内容へ書き換え
aws ecs describe-task-definition --task-definition tomario-staging-task \
  --query 'taskDefinition' > /tmp/broken-task-def.json

python3 -c "
import json
td = json.load(open('/tmp/broken-task-def.json'))
td['containerDefinitions'][0]['command'] = ['this-command-does-not-exist']
for k in ['taskDefinitionArn','revision','status','requiresAttributes','compatibilities','registeredAt','registeredBy']:
    td.pop(k, None)
json.dump(td, open('/tmp/broken-task-def-clean.json','w'))
print('written')
"

aws ecs register-task-definition --cli-input-json file:///tmp/broken-task-def-clean.json \
  --query 'taskDefinition.{family:family,revision:revision}'

# 壊れたリビジョンへデプロイ
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --task-definition tomario-staging-task --force-new-deployment \
  --query 'service.deployments[0].{status:status,taskDef:taskDefinition}'
date   # デプロイ開始時刻

# デプロイ状態を監視（Ctrl+C で終了）
watch -n 10 'aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].deployments[*].{status:status,taskDef:taskDefinition,rolloutState:rolloutState,reason:rolloutStateReason}"'

# ロールバック完了後：冗長数とイベントログを確認
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{desired:desiredCount,running:runningCount}"
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].events[0:10].{time:createdAt,msg:message}" --output table
```

確認するもの：`rolloutState` が `FAILED` → PRIMARY デプロイが元のタスク定義に戻る、`running` が 0 になる瞬間が無い、
FAILED 検知時刻とロールバック完了時刻の差（失敗検知からロールバック完了までの時間）。

### 手順 2（A-02）：タスク強制停止からの自動復旧

```bash
TASK_ARN=$(aws ecs list-tasks --cluster tomario-staging-cluster \
  --service-name tomario-staging-service --desired-status RUNNING --query 'taskArns[0]' --output text)
aws ecs stop-task --cluster tomario-staging-cluster --task $TASK_ARN --reason "障害試験：強制停止テスト"
date   # 停止時刻

watch -n 5 'aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{desired:desiredCount,running:runningCount,pending:pendingCount}"'
# running:2 復帰で Ctrl+C、その時刻をメモ

TG_ARN=$(aws elbv2 describe-target-groups --names tomario-staging-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[*].{IP:Target.Id,Health:TargetHealth.State}"
```

### 手順 3（A-03）：処理中リクエストへの影響

```
【ターミナル】ログイン Cookie 取得 → 予約作成 API へ 100 リクエストのループ ─┐ 同時進行
【別のターミナル】ループ実行中に stop-task ──────────────────────────────────┘
```

```bash
# 【ターミナル】
curl -c cookies.txt -X POST https://{CLOUDFRONT_DOMAIN}/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"loadtest_user_0@example.com","password":"loadtest-password-not-for-prod"}'

for i in $(seq 1 100); do
  STATUS=$(curl -s -o /tmp/booking_response_$i.json -w "%{http_code}" \
    -X POST https://{CLOUDFRONT_DOMAIN}/api/bookings \
    -H "Content-Type: application/json" -b cookies.txt \
    -d '{"room_id":1,"check_in_date":"2026-09-01","check_out_date":"2026-09-03"}')
  echo "リクエスト$i: HTTP $STATUS"; sleep 0.3
done

# 【別のターミナル】ループ中に実行
TASK_ARN=$(aws ecs list-tasks --cluster tomario-staging-cluster \
  --service-name tomario-staging-service --desired-status RUNNING --query 'taskArns[0]' --output text)
aws ecs stop-task --cluster tomario-staging-cluster --task $TASK_ARN --reason "障害試験：処理中リクエストへの影響確認"
```

集計：画面に出た `HTTP xxx` をコード別に数える。5xx のレスポンス本文（`/tmp/booking_response_*.json`）を確認。
DB 確認（ECS Exec 経由、`backup-test-procedure.md` の手順 2 と同じ方法でコンテナに入る）：

```sql
SELECT id, user_id, room_id, total_price, status, created_at
FROM bookings ORDER BY created_at DESC LIMIT 20;
```

`total_price` が NULL/0 の行や `room_id` 不正の行が無いことを確認する。
`modules/backend/alb.tf` の `deregistration_delay = 30` により瞬間的なエラーは想定内。0 件を狙うのではなく「不完全なデータが残らないこと」を主眼にする。

### 手順 4（A-04）：正常なローリングデプロイ中の無停止性

```
【ターミナル】/health へ 0.3 秒間隔の curl ループ ─┐ 同時進行
【別のターミナル】正常なタスク定義のまま force-new-deployment ─┘
```

```bash
# 【ターミナル】デプロイ中ずっと回し続ける
for i in $(seq 1 400); do
  echo "$(date +%T) $(curl -s -o /dev/null -w '%{http_code}' https://{CLOUDFRONT_DOMAIN}/health)"
  sleep 0.3
done | tee /tmp/rolling_deploy_health.log

# 【別のターミナル】ループ開始後に実行（タスク定義は現行のまま = 正常デプロイ）
date   # デプロイ開始時刻
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --force-new-deployment \
  --query "service.deployments[*].{status:status,rolloutState:rolloutState}"
watch -n 5 'aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{running:runningCount,deployments:length(deployments)}"'
# running が 0 にならないことを確認。deployments が 1 になったら Ctrl+C し、その時刻をメモ（安定時刻）

# 【ターミナル】ループ終了後に集計
sort /tmp/rolling_deploy_health.log | awk '{print $2}' | sort | uniq -c
```

確認するもの：`200` 以外（5xx）がほぼ無いこと（数件以内）、デプロイ中も `running` が 0 にならないこと、
`deployments` が最終的に PRIMARY 1 本へ収束すること。

## 実施後の記録

- 結果を [../results/availability-test-result.md](../results/availability-test-result.md) の結果表へ転記する
- A-01 の失敗検知からロールバック完了までの時間、A-02 の復旧時間、A-03 のステータスコード内訳、A-04 のデプロイ中 5xx 件数と所要時間を記録する
- `describe-services` のイベントログ、デプロイ状態遷移、ALB ターゲットヘルス、`rolling_deploy_health.log` を `../evidence/availability/` に格納する
