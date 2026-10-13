# 非機能試験（運用：ロールバック）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の運用試験（O-01・O-02）
対応する結果報告書：[../results/operations-test-result.md](../results/operations-test-result.md)

## 試験実施目的

リリース前には「前に進める手順（デプロイ・構築）」だけでなく「戻す手順」も検証しておく必要がある。
不具合のあるリリースを、直前の正常バージョン（1 つ前の digest）へ手動でロールバックできること・その所要時間を実測で確認する。

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging（`tomario-staging-*`）。O-01 は production の promote フローにも読み替え可能 |
| 起動状態 | cost-start 済みで ECS サービス稼働中であること |
| デプロイ経路 | staging→production は digest promote（再ビルドせず pull→re-tag→push→タスク定義更新） |
| 既知事項 | cost-stop の destroy 連鎖で CloudFront も再作成対象になり、cost-start で CloudFront の再作成に 20〜30 分かかることがある（AWS 側の特性。運用メモ、試験項目としては扱わない） |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| O-01 | デプロイロールバック手順の実演 | staging | 詳細手順 1。現行タスク定義リビジョンと 1 つ前を確認 → `update-service --task-definition <family>:<前のrevision> --force-new-deployment` → `wait services-stable` → 疎通確認。開始〜安定までの時間を記録 → 確認後、最新リビジョンへ戻す | 1 つ前のイメージでサービスが安定し、`/health` と主要 API が正常応答。ロールバック中も `running` が 0 にならない。最新リビジョンへ戻したあと、サービスが安定する | promote フローの切り戻しに相当 |
| O-02 | WAF 緊急デタッチ手順 | production | 詳細手順 2。誤検知で業務影響が出た想定で、Terraform を待たず CLI で Web ACL の関連付けを解除 → 直後にアクセスが復旧することを確認 → 関連付けを戻す | デタッチ実行から数分でアクセスが復旧する。手順書（ランブック）として所要時間を記録 | 誤検知時の初動。P-03（性能）のレイテンシ比較でも同じ手順を使う |

## 詳細手順

### 手順 1（O-01）：デプロイロールバック

```bash
# 現行のサービスが使っているタスク定義と、登録済みリビジョン一覧を確認
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{current:taskDefinition,desired:desiredCount,running:runningCount}"

aws ecs list-task-definitions --family-prefix tomario-staging-task --sort DESC --max-items 5 \
  --query "taskDefinitionArns"

# 1 つ前の正常リビジョンへ切り戻し（<N> は現行 -1 の revision 番号。bootstrapプレースホルダー画像のrevisionは使えないので
# describe-task-definitionでイメージを確認してから選ぶこと）
date   # ロールバック開始時刻
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --task-definition tomario-staging-task:<N> --force-new-deployment \
  --query "service.deployments[*].{status:status,taskDef:taskDefinition,rolloutState:rolloutState}"

aws ecs wait services-stable --cluster tomario-staging-cluster --services tomario-staging-service
date   # 安定時刻 → 差分がロールバック所要時間

# 疎通確認
CF=$(aws cloudfront list-distributions --query "DistributionList.Items[*].{D:DomainName,C:Comment}" --output text | grep staging)
curl -s -o /dev/null -w "%{http_code}\n" "https://<CLOUDFRONT_DOMAIN>/health"
curl -s -o /dev/null -w "%{http_code}\n" "https://<CLOUDFRONT_DOMAIN>/api/rooms?check_in=2026-08-01&check_out=2026-08-03"

# 確認後、最新リビジョンへ戻す（<最新> はロールバック前の revision 番号）
aws ecs update-service --cluster tomario-staging-cluster --service tomario-staging-service \
  --task-definition tomario-staging-task:<最新> --force-new-deployment \
  --query "service.deployments[*].{status:status,taskDef:taskDefinition,rolloutState:rolloutState}"
aws ecs wait services-stable --cluster tomario-staging-cluster --services tomario-staging-service
```

> production の promote フローで切り戻す場合は、`tomario-production-app` の 1 つ前の digest でタスク定義を再登録し、
> production の ECS サービスをそのリビジョンへ `update-service` する（再ビルドはしない）。

確認するもの：`rolloutState` が `COMPLETED` になる、`running` が 0 にならない、`/health` と主要 API が 200、開始〜安定の所要時間。

### 手順 2（O-02）：WAF 緊急デタッチ手順（ランブック）

CloudFront の Web ACL は `wafv2 associate-web-acl` / `disassociate-web-acl`（REGIONAL スコープ専用）では外せない。
ディストリビューション設定の `WebACLId` を書き換えて `update-distribution` する。

```bash
DIST_ID=<CloudFront Distribution ID>

# 1. 現在の設定と ETag を取得し、戻すとき用に WebACLId を控える
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/dist-config.json
ETAG=$(jq -r '.ETag' /tmp/dist-config.json)
ORIGINAL_WEBACL=$(jq -r '.DistributionConfig.WebACLId' /tmp/dist-config.json)

# 2. WebACLId を空にした設定でデタッチ（CloudFront は差分でなく設定全体を送り直す）
jq '.DistributionConfig.WebACLId = ""' /tmp/dist-config.json | jq '.DistributionConfig' > /tmp/dist-config-detached.json
date   # デタッチ開始時刻
aws cloudfront update-distribution --id $DIST_ID --distribution-config file:///tmp/dist-config-detached.json --if-match $ETAG
aws cloudfront wait distribution-deployed --id $DIST_ID

# 3. 誤検知でブロックされていたリクエストが通るようになったか確認
curl -s -o /dev/null -w "%{http_code}\n" "https://<CLOUDFRONT_DOMAIN>/api/rooms?check_in=2026-08-01&check_out=2026-08-03"
date   # 復旧確認時刻 → 差分が復旧所要時間

# 4. 対応完了後、関連付けを戻す（2 の更新で ETag が変わっているため取り直す）
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/dist-config2.json
ETAG2=$(jq -r '.ETag' /tmp/dist-config2.json)
jq --arg acl "$ORIGINAL_WEBACL" '.DistributionConfig.WebACLId = $acl' /tmp/dist-config2.json | jq '.DistributionConfig' > /tmp/dist-config-reattached.json
aws cloudfront update-distribution --id $DIST_ID --distribution-config file:///tmp/dist-config-reattached.json --if-match $ETAG2
aws cloudfront wait distribution-deployed --id $DIST_ID
date   # 再アタッチ完了時刻
```

確認するもの：デタッチから復旧までの所要時間、その間 Terraform state と実体が乖離すること（次の apply で戻る）を認識した上で、ランブックとして手順・所要時間を記録する。

## 実施後の記録

- 結果を [../results/operations-test-result.md](../results/operations-test-result.md) の結果表へ転記する
- O-01 のロールバック所要時間を記録する
- `describe-services` の状態遷移、疎通確認結果を `../evidence/operations/` に格納する
