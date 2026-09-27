# 一部合格（🔺）項目の残作業まとめ

`non-functional-test/results/*.md`で「🔺一部合格」判定の項目だけを抜き出し、完了させるための手順を集約したもの。
「不合格」「未実施」「スキップ」は対象外（それぞれ`staging-remaining-test-items.md`・`production-only-test-items.md`や各resultファイルを参照）。

- 集計日：2026-09-27
- 結果の記入先：`../non-functional-test/results/*.md`（本ファイルは手順集約用）

**注（2026-09-27）**：O-02（cost-stop/cost-startによるインフラ再構築確認）は、日常運用で繰り返し実施しており試験項目として該当しないと判断し除外した（旧O-04と同じ理由）。運用試験の項番はO-01〜O-03に繰り上げ（詳細は`operations-test-result.md`・`test-plan.md`参照）。

## 一覧

| 項番 | 項目 | 対象環境 | 何が足りないか | ステータス |
|---|---|---|---|---|
| M-03 | 閾値超過→メール通知到達 | staging/共通 | 手動でアラーム状態は作ったが、閾値を実際に超過させての発報は未 | ⬜ 要staging起動 |
| M-06 / P-07 | メトリクスダッシュボードの視認性 | staging | RunningTaskCountがContainer Insights未有効のため未表示 | ⬜ 要判断（コスト増） |
| S-04 | ネットワーク境界の構成確認 | staging | (a)(c)(d)確認済み。(b)RDS:3306到達不可のみ残り | 🔺 残りは(b)のみ |

---

## M-03：閾値超過→メール通知到達

**前提**：staging起動中（cost-start済み、ECSがCPUUtilizationメトリクスを発行している状態）であること。cost-stop中は`treat-missing-data: notBreaching`のためどれだけ閾値を下げても発火しない。

```bash
# 現在の設定を全部控える（元に戻す時に使う）
aws cloudwatch describe-alarms --alarm-names tomario-staging-ecs-cpu --output json

# 閾値を一時的に下げて確実に超過させる（他の属性は現状維持のまま上書き）
aws cloudwatch put-metric-alarm \
  --alarm-name tomario-staging-ecs-cpu \
  --alarm-description "ECSサービスのCPU使用率が80%以上になっています" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --dimensions Name=ClusterName,Value=tomario-staging-cluster Name=ServiceName,Value=tomario-staging-service \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 0.01 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --ok-actions arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm \
  --alarm-actions arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm

# 5分周期×2回（最大10分程度）待ってから状態確認
aws cloudwatch describe-alarms --alarm-names tomario-staging-ecs-cpu --query "MetricAlarms[0].StateValue"

# メール（masashi00aws@gmail.com）にALARM通知が届いたか確認

# 確認後、必ず元の閾値(80%)へ戻す
aws cloudwatch put-metric-alarm \
  --alarm-name tomario-staging-ecs-cpu \
  --alarm-description "ECSサービスのCPU使用率が80%以上になっています" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --dimensions Name=ClusterName,Value=tomario-staging-cluster Name=ServiceName,Value=tomario-staging-service \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80.0 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --ok-actions arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm \
  --alarm-actions arn:aws:sns:ap-northeast-1:418295697340:tomario-staging-alarm
```

**確認するもの**：`OK→ALARM`遷移、メール通知の受信。**戻し忘れ厳禁**（閾値0.01%のままだと平常時も常時ALARM状態になる）。

---

## M-06 / P-07：RunningTaskCountダッシュボード表示

**決定（2026-09-27）**：Container Insightsを導入する。試算の結果、実際の運用パターン（cost-start/stopで必要な時だけ起動、月10時間程度）ではメトリクス課金が時間按分のため月10〜15セント程度と無視できる水準と判断（詳細は本ファイルの過去のやり取り・`monitoring-test-result.md`参照）。

**やること**：
- `tomario-infra`のECSクラスター定義（`modules/backend`）に`containerInsights = "enabled"`を追加
- `modules/monitoring`のダッシュボードウィジェットを、`AWS/ECS`ではなく`ECS/ContainerInsights`名前空間の`RunningTaskCount`を参照するよう修正
- dev/staging/production共通モジュールのため、全環境に影響する点に留意
- 導入後、再度ダッシュボードでRunningTaskCountが表示されることを確認してM-06/P-07を合格に格上げする

---

## S-03：⬜スキップ済み（2026-09-27）

将来的に独自ドメインを取得する予定のため、その際にACM証明書へ切替えてまとめて対応する方針に決定。`remaining-task.md` #21に起票済み、一覧から除外。

---

## S-04：ネットワーク境界のまとめ確認

(a)(c)(d)は確認済み（2026-09-27、(d)は`tomario-staging-logs-418295697340`・`tomario-staging-frontend`とも4項目すべて`true`）。残り(b)のみ:

```bash
# (b) ローカルからRDS:3306がタイムアウト/拒否
RDS_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier tomario-staging-rds \
  --query "DBInstances[0].Endpoint.Address" --output text)
nc -zv "$RDS_ENDPOINT" 3306 -w 5   # → timed out / refused を期待
```

RDSはstaging起動中でないと`describe-db-instances`のエンドポイントが引けても接続確認自体は無意味になる点に注意（そもそも到達不可を確認したいので、停止中でも「到達不可」は成立するが、意味のある確認にするなら起動中に実施）。

---

## S-05：✅合格済み（2026-09-27、nonprod・production両方確認完了）

production確認には`aws --profile tomario-prod`（アカウント236782813946、`OrganizationAccountAccessRole`）でアクセスできることが判明。ECS/ALB起動状態と無関係に確認可能だった（GuardDutyはアカウント単位、S3配信は常時発生、CloudTrailイベントは"直近1時間"ではなく過去に遡って検索すれば見つかる）。詳細は`security-test-result.md`参照。

---

## S-07：✅合格に格上げ済み（2026-09-27）

`security-test-result.md`が2026-09-16時点の`test-summary.md`の格上げ判断に追従できていなかった（反映漏れ）ため訂正。対応完了、一覧から除外。
