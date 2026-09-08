# 非機能試験（運用：ロールバック・インフラ再構築）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の運用試験（O-01・O-02・O-04・O-05）
対応する結果報告書：[../results/operations-test-result.md](../results/operations-test-result.md)

## 試験実施目的

リリース前には「前に進める手順（デプロイ・構築）」だけでなく「戻す手順・作り直す手順」も検証しておく必要がある。
以下を実測で確認する。

- 不具合のあるリリースを、直前の正常バージョン（1 つ前の digest）へ手動でロールバックできること・その所要時間
- cost-stop で destroy されたインフラを、cost-start（Terraform apply）で正しく再構築できること

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging（`tomario-staging-*`）。O-01 は production の promote フローにも読み替え可能 |
| 起動状態 | O-01 は cost-start 済みで ECS サービス稼働中。O-02 は cost-stop 済みの状態から開始 |
| デプロイ経路 | staging→production は digest promote（再ビルドせず pull→re-tag→push→タスク定義更新） |
| 既知事項 | cost-stop の destroy 連鎖で CloudFront も再作成対象になり、cost-start で CloudFront の再作成に 20〜30 分かかることがある（AWS 側の特性） |
| 注意 | O-01 でロールバック後は、確認が済んだら最新リビジョンへ戻す（または戻さない判断を記録する） |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| O-01 | デプロイロールバック手順の実演 | staging | 詳細手順 1。現行タスク定義リビジョンと 1 つ前を確認 → `update-service --task-definition <family>:<前のrevision> --force-new-deployment` → `wait services-stable` → 疎通確認。開始〜安定までの時間を記録 | 1 つ前のイメージでサービスが安定し、`/health` と主要 API が正常応答。ロールバック中も `running` が 0 にならない | promote フローの切り戻しに相当。所要時間を面接で定量的に語れるようにする |
| O-02 | cost-stop / cost-start によるインフラ再構築確認 | staging | 詳細手順 2。cost-stop 実行後の状態を `terraform plan`（`N to add`）で確認 → cost-start（`account_group` / `env` を指定）実行 → 完了後に全リソースの存在と CloudFront 経由の疎通を確認 | destroy されていた ALB・ECS サービス・VPC エンドポイント・CloudWatch アラーム等が Terraform で再作成され、`terraform plan` が差分ゼロ。CloudFront 経由でフロント・API が疎通 | 日常運用で実施している内容の記録。State と実体の整合が取れていることの確認も兼ねる |
| O-04 | WAF Web ACL の cost-stop/start 組み込み | production（未公開期間） | 詳細手順 3。cost-start（`account_group=prod, env=production`）で Web ACL 作成＋CloudFront への関連付けが行われることを確認 → `terraform plan` 差分ゼロ → cost-stop で関連付け解除＋Web ACL 削除が行われ、`wafv2 list-web-acls` から消えることを確認 | cost-start 後：WAF が CloudFront に関連付き、`terraform plan` 差分ゼロ。cost-stop 後：Web ACL が削除され課金が止まる。CloudFront の destroy 連鎖に巻き込まれないこと | `cost-stop.yml` / `cost-start.yml` への組み込みは production 構築時に実装（`security-environment-design.md`） |
| O-05 | WAF 緊急デタッチ手順 | production（未公開期間） | 詳細手順 4。誤検知で業務影響が出た想定で、Terraform を待たず CLI で Web ACL の関連付けを解除 → 直後にアクセスが復旧することを確認 → 関連付けを戻す | `disassociate-web-acl` 実行から数分でアクセスが復旧する。手順書（ランブック）として所要時間を記録 | 誤検知時の初動。P-09 のレイテンシ比較でも同じ手順を使う |
| O-03 | ロールバック後の復帰（後始末） | staging | 確認完了後、最新リビジョンへ `update-service` で戻す（または「戻さず様子見」の判断を記録） | サービスが最新リビジョンで安定、または判断が記録される | O-01 の後始末 |

## 詳細手順

### 手順 1（O-01）：デプロイロールバック

```bash
# 現行のサービスが使っているタスク定義と、登録済みリビジョン一覧を確認
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{current:taskDefinition,desired:desiredCount,running:runningCount}"

aws ecs list-task-definitions --family-prefix tomario-staging-task --sort DESC --max-items 5 \
  --query "taskDefinitionArns"

# 1 つ前の正常リビジョンへ切り戻し（<N> は現行 -1 の revision 番号）
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
```

> production の promote フローで切り戻す場合は、`tomario-production-app` の 1 つ前の digest でタスク定義を再登録し、
> production の ECS サービスをそのリビジョンへ `update-service` する（再ビルドはしない）。

確認するもの：`rolloutState` が `COMPLETED` になる、`running` が 0 にならない、`/health` と主要 API が 200、開始〜安定の所要時間。

### 手順 2（O-02）：インフラ再構築

```bash
# cost-stop 済みの状態を確認（envs/nonprod/staging 等の作業ディレクトリで）
terraform plan   # "N to add" と表示され、destroy 済みリソースが再作成対象になっていること

# cost-start ワークフローを実行（GitHub Actions）
#   Actions → cost-start → Run workflow → account_group / env を指定
#   （RDS 起動 → terraform apply → ECS スケール の順で実行される）

# 完了後の確認
terraform plan   # → No changes（差分ゼロ）
aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{desired:desiredCount,running:runningCount}"
aws elbv2 describe-load-balancers --names tomario-staging-alb --query "LoadBalancers[0].State.Code"   # → active
curl -s -o /dev/null -w "%{http_code}\n" "https://<CLOUDFRONT_DOMAIN>/health"
```

確認するもの：`terraform plan` が差分ゼロ、ALB が `active`、ECS が `running=2`、CloudFront 経由で疎通。
CloudFront を道連れ再作成した場合は反映に 20〜30 分かかることを記録する。

### 手順 3（O-04）：WAF Web ACL の cost-stop/start 組み込み

```bash
REGION=us-east-1   # CloudFront スコープの WAF は us-east-1
DIST_ID=<CloudFront Distribution ID>

# cost-start 後：Web ACL が作成され CloudFront に関連付いているか
aws wafv2 list-web-acls --scope CLOUDFRONT --region $REGION \
  --query "WebACLs[*].{Name:Name,ARN:ARN}"
aws wafv2 get-web-acl-for-resource \
  --resource-arn arn:aws:cloudfront::<account>:distribution/$DIST_ID --region $REGION \
  --query "WebACL.Name"
terraform plan   # → No changes

# cost-stop 後：関連付け解除 + Web ACL 削除が行われたか
aws wafv2 list-web-acls --scope CLOUDFRONT --region $REGION --query "WebACLs[*].Name"   # → 対象が消えている
```

確認するもの：cost-start/stop のたびに Web ACL の作成・関連付け・解除・削除が例外なく回ること、`terraform plan` 差分ゼロ、CloudFront の destroy 連鎖に WAF が悪影響しないこと。

### 手順 4（O-05）：WAF 緊急デタッチ手順（ランブック）

```bash
REGION=us-east-1
DIST_ID=<CloudFront Distribution ID>
RES_ARN=arn:aws:cloudfront::<account>:distribution/$DIST_ID

WEBACL_ARN=$(aws wafv2 get-web-acl-for-resource --resource-arn $RES_ARN --region $REGION \
  --query "WebACL.ARN" --output text)   # 戻すとき用に控える
date   # デタッチ開始時刻
aws wafv2 disassociate-web-acl --resource-arn $RES_ARN --region $REGION

# CloudFront への反映を待ち、誤検知でブロックされていたリクエストが通るようになったか確認
curl -s -o /dev/null -w "%{http_code}\n" "https://<CLOUDFRONT_DOMAIN>/api/rooms?check_in=2026-08-01&check_out=2026-08-03"
date   # 復旧確認時刻 → 差分が復旧所要時間

# 対応完了後、関連付けを戻す
aws wafv2 associate-web-acl --web-acl-arn $WEBACL_ARN --resource-arn $RES_ARN --region $REGION
```

確認するもの：デタッチから復旧までの所要時間、その間 Terraform state と実体が乖離すること（次の apply で戻る）を認識した上で、ランブックとして手順・所要時間を記録する。

## 実施後の記録

- 結果を [../results/operations-test-result.md](../results/operations-test-result.md) の結果表へ転記する
- O-01 のロールバック所要時間、O-02 の cost-start 所要時間（CloudFront 再作成の有無）を記録する
- `describe-services` の状態遷移、`terraform plan` の出力（差分ゼロ）、疎通確認結果を `../evidence/operations/` に格納する
- **O-03（最新リビジョンへ戻した／様子見の判断）をチェックする**
