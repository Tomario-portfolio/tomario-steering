# 障害対応手順（ランブック）

最終更新：2026-10-13

平常時に障害を検知したときの対応手順。リリース直後の集中監視期間は[ハイパーケア・運用引き継ぎ体制](../release-management/hypercare-and-handover.md)、リリースの切り戻しは[カットオーバー手順・切り戻し手順](../release-management/cutover-and-rollback.md)を参照する。

- 対象：production（stagingも同じ手順で読み替えられる。環境名の`production`を`staging`に置き換える）
- 体制：判断者・対応者・記録者を1人で兼務する。手順の中の「判断」は自分で行う
- コマンドは読み取りのみのものを先に並べている。変更を伴う操作には「変更」と明記する

---

## 1. 対応の流れ

| 段階 | やること | 目安 |
|------|---------|------|
| 1. 検知 | アラームのメール（SNS）、または利用者からの報告で気づく | ― |
| 2. 状況確認 | 「2. 最初に確認すること」で、影響の有無と範囲をつかむ | 5分以内 |
| 3. 判断 | 「4. 対応方針の判断」で、静観・切り戻し・修正・復元のどれにするか決める | 状況確認の直後 |
| 4. 対応 | 「3. アラーム別の初動」の手順で対応する | ― |
| 5. 記録 | 「5. 記録と振り返り」に従って記録する。サービスに影響が出た場合は振り返りも行う | 当日中 |

---

## 2. 最初に確認すること

アラームの種類に関係なく、まず以下で「利用者に影響が出ているか」をつかむ。

```bash
export AWS_PROFILE=tomario-prod   # stagingの場合は不要
ENV=production

# 1. 外から見て動いているか（CloudFront経由）
curl -s -o /dev/null -w "%{http_code}\n" "https://<CloudFrontドメイン>/health"

# 2. 鳴っているアラームの一覧（WAFのアラームはus-east-1にあるため別に見る）
aws cloudwatch describe-alarms --alarm-name-prefix "tomario-${ENV}" --state-value ALARM \
  --query "MetricAlarms[*].{Name:AlarmName,Reason:StateReason}" --output table
aws cloudwatch describe-alarms --alarm-name-prefix "tomario-${ENV}-waf" --region us-east-1 \
  --query "MetricAlarms[*].{Name:AlarmName,State:StateValue}" --output table

# 3. ECSサービスの状態（タスク数・デプロイ中か）と直近のイベント
aws ecs describe-services --cluster tomario-${ENV}-cluster --services tomario-${ENV}-service \
  --query "services[0].{desired:desiredCount,running:runningCount,pending:pendingCount,deployments:deployments[*].{status:status,rollout:rolloutState,taskDef:taskDefinition}}"
aws ecs describe-services --cluster tomario-${ENV}-cluster --services tomario-${ENV}-service \
  --query "services[0].events[0:10].{time:createdAt,msg:message}" --output table

# 4. ALBのターゲットの状態
TG_ARN=$(aws elbv2 describe-target-groups --names tomario-${ENV}-tg --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[*].{IP:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}"

# 5. 直近30分のアプリのエラーログ
aws logs filter-log-events --log-group-name /ecs/tomario-${ENV} \
  --start-time $(( ($(date +%s) - 1800) * 1000 )) \
  --filter-pattern '?ERROR ?Traceback ?" 500 "' \
  --query "events[*].message" --output text | tail -50
```

あわせて、直近のデプロイの有無を確認する（`tomario-app`・`tomario-infra`のGitHub Actionsの実行履歴）。**直近に変更を入れていれば、まずその変更を疑う。**

AWS側の障害が疑われる場合は、AWS Health Dashboard（コンソール）で東京リージョンの障害情報を確認する。

---

## 3. アラーム別の初動

### 3.1 `tomario-{env}-alb-unhealthy-host`（ALBのターゲットに異常）

ALBのヘルスチェック（`/health`）に失敗しているタスクがある。タスクは2台あるため、1台だけなら利用者への影響は小さい。

1. 「2. 最初に確認すること」の3・4で、異常なタスクの数と、ECSが自動で入れ替え中かを確認する
   - ECSは異常なタスクを自動で停止・再起動する（[A-02](../verification/non-functional-test/results/availability-test-result.md)で、1分未満で2台に戻ることを確認済み）。数分で`healthy`に戻れば静観してよい
2. 戻らない場合は、停止したタスクの停止理由を確認する

   ```bash
   STOPPED=$(aws ecs list-tasks --cluster tomario-${ENV}-cluster --desired-status STOPPED --query "taskArns[0:5]" --output text)
   aws ecs describe-tasks --cluster tomario-${ENV}-cluster --tasks $STOPPED \
     --query "tasks[*].{stopped:stoppedAt,reason:stoppedReason,exit:containers[0].exitCode,detail:containers[0].reason}"
   ```

3. 原因ごとの対応
   - **デプロイの直後** → デプロイサーキットブレーカーが自動で直前のリビジョンに戻す（[A-01](../verification/non-functional-test/results/availability-test-result.md)）。デプロイが完了したあとに異常が出た場合は、[切り戻し手順 2.2](../release-management/cutover-and-rollback.md)で1つ前のリビジョンに戻す（変更）
   - **ログにDB接続のエラー**（`OperationalError`等） → 「3.3」のRDSの確認へ
   - **イメージを取得できない・シークレットを取得できない**（`CannotPullContainerError`、`ResourceInitializationError`） → VPCエンドポイント（ECR・Secrets Manager・CloudWatch Logs）とタスク実行ロールを確認する

### 3.2 `tomario-{env}-ecs-cpu`（ECSのCPU使用率 80%以上が10分）

1. Auto Scalingが働いているかを確認する（min 2 / max 4、目標CPU 70%）

   ```bash
   aws application-autoscaling describe-scaling-activities --service-namespace ecs \
     --resource-id service/tomario-${ENV}-cluster/tomario-${ENV}-service \
     --query "ScalingActivities[0:5].{Time:StartTime,Desc:Description,Status:StatusCode}" --output table
   ```

   - タスク数が増えていて、CPUが下がってきていれば静観する（スケールアウトは約3分で反映される。[P-01](../verification/non-functional-test/results/performance-test-result.md)）
   - アクティビティが`Failed`なら、Descriptionを確認する（IAM権限不足など）
2. max（4台）に達してもCPUが下がらない場合は、アクセスの中身を確認する
   - 特定のIPからの大量アクセスや攻撃であれば、WAFのログ（「3.4」）とALBアクセスログ（S3の`/alb/`）で送信元を確認する
   - 正規のアクセスの増加であれば、`tomario-infra`の`autoscaling_max_capacity`を引き上げる（変更）
3. ログインはパスワード照合（PBKDF2）でCPUを多く使う。ログインの急増でもCPUが上がる

### 3.3 `tomario-{env}-rds-cpu`（RDSのCPU使用率 80%以上が10分）

1. 同じ時間帯のECSのCPU・タスク数と、アプリのログ（遅いリクエスト・エラー）を確認する。アクセス増加に伴うものか、RDS単独の問題かを切り分ける
2. RDSの状態と直近のイベントを確認する

   ```bash
   aws rds describe-db-instances --db-instance-identifier tomario-${ENV}-rds \
     --query "DBInstances[0].{status:DBInstanceStatus,class:DBInstanceClass,az:AvailabilityZone}"
   aws rds describe-events --source-identifier tomario-${ENV}-rds --source-type db-instance --duration 120 \
     --query "Events[*].{time:Date,msg:Message}" --output table
   ```

3. 一時的な負荷で、利用者への影響（応答の遅れ・エラー）が続く場合は、インスタンスクラスを一時的に上げて逃がす（変更。[性能試験の手順書](../verification/non-functional-test/procedures/performance-test-procedure.md)の詳細手順1-2と同じ操作。変更中は数分間接続できない）
4. **データの破損・誤削除**が原因であれば、ポイントインタイムリストアで復旧する（変更）。手順は[バックアップ試験の手順書](../verification/non-functional-test/procedures/backup-test-procedure.md)のB-02を参照する。復元後にアプリの接続先を切り替える手順は、production用の復元手順として別途作成する

RDSはSingle-AZのため、AZの障害時に自動のフェイルオーバーは無い（[ADR](../adr/infra/database/003-multi-az-cost-tradeoff.md)）。AWS Health DashboardでAZの障害と分かった場合は、復旧を待つか、別のAZへのポイントインタイムリストアを判断する。

### 3.4 `tomario-{env}-waf-blocked`（WAFのブロック数 10件超／5分）

productionでセキュリティスタック（WAF・AWS Config・Security Hub）を有効にしているときだけ存在するアラーム。

WAFが攻撃とみなしたリクエストをブロックしている。**ブロックできているので、それ自体は障害ではない。** 確認すべきは「正規の利用者がブロックされていないか（誤検知）」。

1. ブロックされたリクエストの中身を確認する（WAFのログはus-east-1）

   ```bash
   QID=$(aws logs start-query --region us-east-1 --log-group-name aws-waf-logs-tomario-production-cloudfront \
     --start-time $(( $(date +%s) - 3600 )) --end-time $(date +%s) \
     --query-string 'fields @timestamp, terminatingRuleId, httpRequest.clientIp, httpRequest.uri | filter action="BLOCK" | stats count(*) by terminatingRuleId, httpRequest.clientIp, httpRequest.uri | sort count(*) desc | limit 20' \
     --query queryId --output text)
   sleep 5; aws logs get-query-results --region us-east-1 --query-id $QID
   ```

2. 判断
   - **攻撃のペイロード・特定のIPからの大量アクセス** → 防御できているので静観し、記録だけ残す
   - **通常の画面操作（予約フォームの送信など）がブロックされている** → 誤検知。利用者への影響が大きければ、[WAF緊急デタッチ手順（O-02）](../verification/non-functional-test/procedures/operations-test-procedure.md)でWAFを一時的に外す（変更）。その後、誤検知したルールを除外する設定を`tomario-infra`に入れてから戻す

---

## 4. 対応方針の判断

| 状況 | 方針 |
|------|------|
| 数分で自然に回復した（ECSの自動復旧・スケールアウト） | 静観。記録のみ |
| 直近のデプロイの後から異常が出ている | 原因の調査より先に、1つ前のリビジョンへ切り戻す（[切り戻し手順 2.2](../release-management/cutover-and-rollback.md)）。利用者への影響を止めることを優先する |
| 直近の変更が無く、原因が設定・コードの不具合と分かった | 修正をデプロイする（通常のCI/CD） |
| データが壊れた・消えた | ポイントインタイムリストアを検討する。戻す時点より後のデータは失われるため、失われる範囲を確認してから判断する |
| AWS側の障害 | AWS Health Dashboardで確認し、復旧を待つ。待てない場合のみ回避策（別AZへの復元など）を検討する |

---

## 5. 記録と振り返り

- **記録**：すべての障害対応を、`tomario-workspace/history/activity_log/`の当日のログに書く。書く内容は、検知の時刻と経路、影響の範囲、行った対応、復旧の時刻
- **振り返り**：productionで利用者に影響が出た場合（エラー・応答の遅れ・データの損失）は、記録に加えて振り返りを書く。書く内容は、時系列、原因、なぜ防げなかったか、再発防止策。再発防止策は`tomario-workspace/task-and-flow/remaining-task.md`に追加する
- 利用者に影響が無かったもの（静観で済んだもの・攻撃のブロック）は、記録のみでよい
- 手順が実際と違った・足りなかった場合は、この文書を直す

## 6. 連絡先

| 用途 | 連絡先 |
|------|-------|
| アラームの通知 | 管理者のメールアドレス（SNSトピック`tomario-production-alarm`、WAFは`tomario-production-waf-alarm`） |
| 対応者 | 管理者本人（1人運用のため、エスカレーション先は無い） |
| AWS側の障害の確認 | AWS Health Dashboard |

---

## 7. 既知の落とし穴

- **アラームが無い障害がある**：アプリが5xxを返す障害（ALBのターゲットは`healthy`のまま、特定のAPIだけエラーになる場合）を検知するアラームは無い。利用者からの報告や画面の確認で気づいた場合は、「2. 最初に確認すること」の5のエラーログから調べる
- **画面のステータスコードを信用しない**：CloudFrontはSPA対応のため、404を200に置き換えて返す。原因はALB・アプリのログで確認する
- **cost-startの直後**
  - アラームは最大10分ほど`INSUFFICIENT_DATA`になる（評価期間 5分×2回）。障害ではない
  - CloudFrontの再作成で、画面が表示されるまで20〜30分かかることがある
  - ECSサービスは`task_definition`を`ignore_changes`にしているため、Terraformで更新したタスク定義は、cost-startでサービスが作り直されたときに初めて反映される
  - 1つ前のタスク定義のリビジョンが、bootstrap用のプレースホルダー（ECRに実体が無いイメージ）になっていることがある。切り戻すときは`describe-task-definition`でイメージを確認してから、実際のイメージが入っている直近のリビジョンを選ぶ
- **RDSの7日自動起動**：停止中のRDSは7日でAWSにより自動起動される。`rds-autostop` Lambdaが1時間ごとに確認し、ECSが稼働していなければ再停止する。ECSを止めたままRDSだけを起動して調査していると、自動起動の直後であればLambdaに止められることがある
- **cost-stop中のproduction**：ALBもタスクも無いため、アラームは`INSUFFICIENT_DATA`になる。障害ではない
- **WAFのアラーム・ログはus-east-1**：CloudFront用のWAFのため、東京リージョンではなくus-east-1で確認する
- **ECS Execで長いコマンドを貼り付けない**：SSMのセッションで文字化けしやすい。パスワード等は1行でファイルに保存し、スクリプト側で読み込む
- **セッション署名鍵のローテーション（90日ごと）**：1つ前の鍵も受け付けるため、ローテーションでログインが切れることは無い。短い間隔で2回続けて更新した場合のみ切れる

## 8. 関連ドキュメント

- [カットオーバー手順・切り戻し手順](../release-management/cutover-and-rollback.md)
- [ハイパーケア・運用引き継ぎ体制](../release-management/hypercare-and-handover.md)
- [運用試験の手順書](../verification/non-functional-test/procedures/operations-test-procedure.md)：デプロイの切り戻し（O-01）・WAF緊急デタッチ（O-02）
- [バックアップ試験の手順書](../verification/non-functional-test/procedures/backup-test-procedure.md)：ポイントインタイムリストア（B-02）
- [監視の環境定義](https://github.com/Tomario-portfolio/tomario-docs/blob/main/environment-definitions/monitoring-environment-design.md)：アラームの閾値・ログの保持期間
