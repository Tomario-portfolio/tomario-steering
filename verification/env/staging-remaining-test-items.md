# staging 未実施の非機能試験項目・手順

`non-functional-test/`の44項目のうち、**staging（および共通環境）で未実施の13項目**をまとめたもの。production固有の10項目は`production-only-test-items.md`を参照。

- 出典：`../non-functional-test/test-plan.md`・`../non-functional-test/procedures/*.md`
- 結果の記入先：`../non-functional-test/results/*.md`（本ファイルは手順集約用）
- 集計日：2026-09-12

## 環境固有値（staging）

| 項目 | 値 |
|---|---|
| AWSアカウント | `418295697340`（nonprod） |
| CloudFront Distribution ID | `E1XHUK5RP1RUM8` |
| CloudFrontドメイン | `https://d14h67xxnvmsdi.cloudfront.net` |
| ECSクラスター/サービス | `tomario-staging-cluster` / `tomario-staging-service` |
| SNSトピック（アラーム通知） | `arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm` |
| RDSインスタンス | `tomario-staging-rds` |

**注意**：staging は`cost-stop`で backend/network が destroy される運用。試験実施前に`cost-start`（`account_group=nonprod, env=staging`）で起動しておくこと。ECS/ALBが無い状態だとM-01/M-02/A-04/A-05/O-01/O-03等はそのまま失敗する。

---

## 一覧

| 項番 | 項目 | 対象 |
|---|---|---|
| P-00 | ベースライン測定 | staging |
| P-08 | 目標スループット達成確認 | staging |
| A-04 | 正常なローリングデプロイ中の無停止性 | staging |
| A-05 | 壊れたリビジョンの後片付け | staging |
| B-06 | 手動スナップショットからの復元（任意） | staging |
| M-01 | SNSサブスクリプション確認 | dev/staging/production共通 |
| M-02 | アラームがOK状態 | dev/staging/production共通 |
| M-04 | ログ追跡性（Logs Insights） | dev/staging/production共通 |
| O-01 | デプロイロールバック手順の実演 | staging |
| O-03 | ロールバック後の復帰（後始末） | staging |
| S-01 | 依存パッケージの脆弱性スキャン | CI（tomario-app） |
| S-02 | コンテナイメージの脆弱性スキャン | ECR/ローカル |
| S-06 | IAM最小権限の棚卸し（任意） | 全環境 |

---

## P-00：ベースライン測定

```bash
# API応答時間（数回計測）
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{time_total}\n" "https://d14h67xxnvmsdi.cloudfront.net/api/rooms?check_in=2026-08-01&check_out=2026-08-03"
done

# ECSのCPU/メモリ使用率（直近30分）
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=tomario-staging-cluster Name=ServiceName,Value=tomario-staging-service \
  --start-time $(date -v-30M +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Average
```

**確認するもの**：負荷なし時の応答時間・CPU/メモリの定常値。P-05・P-08の評価基準として記録しておく。

---

## P-08：目標スループット達成確認

前提：非機能要件定義書に目標rps（およびp95許容値）が定義されていること。未定義なら暫定値を置いて実測し、結果を要件へフィードバックする。

```bash
# 目標rpsに合わせてk6のstagesを調整して実行（loadtest-throughput.js等）
k6 run loadtest-throughput.js
```

**確認するもの**：目標rpsを維持した状態で成功率≥99%、p(95)が許容内。`http_reqs`のrps・`checks_succeeded`・`http_req_duration`p(95)を取得。

---

## A-04：正常なローリングデプロイ中の無停止性

```bash
# 【ターミナルA】デプロイ中ずっと回し続ける
for i in $(seq 1 400); do
  echo "$(date +%T) $(curl -s -o /dev/null -w '%{http_code}' https://d14h67xxnvmsdi.cloudfront.net/health)"
  sleep 0.3
done | tee /tmp/rolling_deploy_health.log

# 【ターミナルB】ループ開始後に実行（タスク定義は現行のまま=正常デプロイ）
date
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --force-new-deployment \
  --query "service.deployments[*].{status:status,rolloutState:rolloutState}"
aws ecs wait services-stable --cluster tomario-staging-cluster --services tomario-staging-service
date

# 【ターミナルA】ループ終了後に集計
sort /tmp/rolling_deploy_health.log | awk '{print $2}' | sort | uniq -c
```

**確認するもの**：`200`以外（5xx）がほぼ無いこと（数件以内）。デプロイ中も`running`が0にならないこと。`deployments`が最終的にPRIMARY1本へ収束すること。

---

## A-05：壊れたリビジョンの後片付け

```bash
aws ecs deregister-task-definition --task-definition tomario-staging-task:<壊れたrevision>
```

**確認するもの**：壊れたリビジョンが`INACTIVE`になる。A-01（デプロイサーキットブレーカー試験）の後始末。

---

## B-06：手動スナップショットからの復元（任意）

```bash
aws rds create-db-snapshot --db-instance-identifier tomario-staging-rds --db-snapshot-identifier tomario-staging-manual-$(date +%Y%m%d)
aws rds wait db-snapshot-available --db-snapshot-identifier tomario-staging-manual-$(date +%Y%m%d)

aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier tomario-staging-restore-test \
  --db-snapshot-identifier tomario-staging-manual-$(date +%Y%m%d)
aws rds wait db-instance-available --db-instance-identifier tomario-staging-restore-test

# 検証後、復元先を削除
aws rds delete-db-instance --db-instance-identifier tomario-staging-restore-test --skip-final-snapshot
```

**確認するもの**：スナップショット取得時点の状態で復元できること。PITRとは別経路の復旧手段の確認。優先度低。

---

## M-01：SNSサブスクリプション確認

```bash
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm \
  --query "Subscriptions[*].{Protocol:Protocol,Endpoint:Endpoint,Arn:SubscriptionArn}"
```

**確認するもの**：`SubscriptionArn`が`PendingConfirmation`ではなく実ARNになっていること。未確認だとアラーム発報しても通知が届かない。

---

## M-02：アラームがOK状態

```bash
aws cloudwatch describe-alarms --alarm-name-prefix "tomario-staging" \
  --query "MetricAlarms[*].{Name:AlarmName,State:StateValue,Reason:StateReason}" --output table
```

**確認するもの**：全アラームの`StateValue`が`OK`（`INSUFFICIENT_DATA`でない）。起動直後は評価期間5分×2回で最大10分`INSUFFICIENT_DATA`になりうる。

---

## M-04：ログ追跡性（CloudWatch Logs Insights）

```bash
aws logs start-query \
  --log-group-name "/ecs/tomario-staging" \
  --start-time $(date -v-1H +%s 2>/dev/null || date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'
# 返るqueryIdをget-query-resultsに渡す
aws logs get-query-results --query-id <queryId>
```

**確認するもの**：エラー行から時刻・エンドポイント・スタックトレースまで辿れること。特定リクエストの追跡は`filter @message like /<request id>/`等で絞り込む。

---

## O-01：デプロイロールバック手順の実演

```bash
# 現行のサービスが使っているタスク定義と、登録済みリビジョン一覧を確認
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{current:taskDefinition,desired:desiredCount,running:runningCount}"

aws ecs list-task-definitions --family-prefix tomario-staging-task --sort DESC --max-items 5 \
  --query "taskDefinitionArns"

# 1つ前の正常リビジョンへ切り戻し（<N>は現行-1のrevision番号）
date
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --task-definition tomario-staging-task:<N> --force-new-deployment \
  --query "service.deployments[*].{status:status,taskDef:taskDefinition,rolloutState:rolloutState}"

aws ecs wait services-stable --cluster tomario-staging-cluster --services tomario-staging-service
date

# 疎通確認
curl -s -o /dev/null -w "%{http_code}\n" "https://d14h67xxnvmsdi.cloudfront.net/health"
curl -s -o /dev/null -w "%{http_code}\n" "https://d14h67xxnvmsdi.cloudfront.net/api/rooms?check_in=2026-08-01&check_out=2026-08-03"
```

**確認するもの**：`rolloutState`が`COMPLETED`になる、`running`が0にならない、`/health`と主要APIが200、開始〜安定の所要時間。

---

## O-03：ロールバック後の復帰（後始末）

O-01確認完了後、最新リビジョンへ`update-service`で戻す（または「戻さず様子見」の判断を記録する）。

---

## S-01：依存パッケージの脆弱性スキャン（CI）

```bash
# ローカル先行実行
cd /path/to/tomario-app
pip-audit -r requirements.txt
```

**確認するもの**：Critical/Highゼロ、または各指摘に対応方針（バージョン更新／許容理由）を記録。`deploy.yml`への`pip-audit`ステップ追加自体が前提（SEC-4、未実装）。

---

## S-02：コンテナイメージの脆弱性スキャン

```bash
# ECRプッシュ時スキャン結果を取得
aws ecr describe-image-scan-findings --repository-name tomario-staging-app --image-id imageTag=<tag>

# またはローカルでTrivy
trivy image <ECRイメージURI>
```

**確認するもの**：Critical/Highゼロ、または対応方針を記録。ベースイメージ更新で解消するものが多い。

---

## S-06：IAM最小権限の棚卸し（任意）

```bash
# 各ロールの未使用アクセス分析（IAM Access Analyzer）
aws accessanalyzer list-findings --analyzer-arn <analyzer ARN>

# または個別ロールの未使用権限確認
aws iam generate-service-last-accessed-details --arn <role ARN>
aws iam get-service-last-accessed-details --job-id <job ID>
```

**確認するもの**：未使用の広範な権限がない、または削減方針を記録。優先度低（職務分掌はインフラ用／デプロイ用で設計済み）。
