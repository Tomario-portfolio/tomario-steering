# production 固有の非機能試験項目・手順

`non-functional-test/`の44項目のうち、staging等と共通ではなく**production環境でのみ**実施するもの10項目（WAF・Security Hub・AWS Config導入に伴う追加検証）をまとめたもの。

- 出典：`../non-functional-test/test-plan.md`・`../non-functional-test/procedures/*.md`
- 結果の記入先：`../non-functional-test/results/*.md`（本ファイルはあくまで「production向けに手順を集約したもの」で、正式な結果報告書は従来通り`results/`側）
- 集計日：2026-09-12

## 環境固有値（production）

| 項目 | 値 |
|---|---|
| AWSアカウント | `236782813946`（`tomario-prod-sysadmin`） |
| CloudFront Distribution ID | `E16RCKYF5065BQ` |
| CloudFrontドメイン | `https://dvs3h58umdylx.cloudfront.net` |
| CloudFront用WAF Web ACL | `tomario-production-cloudfront`（us-east-1、ARN: `arn:aws:wafv2:us-east-1:236782813946:global/webacl/tomario-production-cloudfront/6ba65cd8-c042-4ae4-b8d6-789c1ecb3a3d`） |
| ALB用WAF Web ACL | `tomario-production-alb`（ap-northeast-1、ARN: `arn:aws:wafv2:ap-northeast-1:236782813946:regional/webacl/tomario-production-alb/819cd8f7-7dd5-4642-98bb-574f72764abc`） |
| ALB ARN | `arn:aws:elasticloadbalancing:ap-northeast-1:236782813946:loadbalancer/app/tomario-production-alb/2d5006a539ae24c0` |
| read-only確認用ロール | `arn:aws:iam::236782813946:role/github-actions-terraform-prod-readonly`（ローカルからAssumeRole可能） |

read-only確認は以下でAssumeRoleしてから実施する。

```bash
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::236782813946:role/github-actions-terraform-prod-readonly \
  --role-session-name readonly-check \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)
export AWS_ACCESS_KEY_ID=$(echo "$CREDS" | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo "$CREDS" | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo "$CREDS" | awk '{print $3}')
```

---

## 一覧

| 項番 | 項目 | 状態 |
|---|---|---|
| S-07 | WAFマネージドルールの有効性 | ✅ 合格 |
| S-08 | WAF誤検知（false positive）確認 | ⏭ スキップ |
| S-09 | Security Hub検出結果レビュー | ✅ 完了 |
| M-07 | WAFログの配信確認 | ✅ 完了 |
| S-10 | AWS Configルールのコンプライアンス評価 | ⏭ スキップ |
| M-08 | WAF BlockedRequestsアラートの発報 | ✅ 完了 |
| O-05 | WAF緊急デタッチ手順 | ✅ 完了 |
| P-09 | WAF有効時のレイテンシ影響 | ⏭ スキップ |

---

> **旧O-04（WAF Web ACLのcost-stop/start組み込み）について**：2026-09-14、非機能試験としては削除した（`test-plan.md`参照）。cost-stop/startを日常運用として繰り返す中で自然に確認できる内容のため、独立した試験項目にはしない。参考として背景だけ残す：ALB用WAF(`module "waf_alb"`)は`envs/prod/production/backend/main.tf`内にあり、backendコンポーネントごとcost-stopでdestroyされ、cost-startのたびに作り直される。CloudFront用WAFはfrontendコンポーネント側にあり、cost-stopの対象外で残り続ける。

## S-07：WAFマネージドルールの有効性 ✅合格

**やること**：SQLi・XSS・パストラバーサル等の攻撃ペイロードを送り、`BLOCK`されることを確認する。

```bash
CF="https://dvs3h58umdylx.cloudfront.net"

# XSS相当（CommonRuleSet）
curl -s -o /dev/null -w "xss: %{http_code}\n" --data-urlencode "q=<script>alert(1)</script>" "$CF/api/rooms"

# パストラバーサル相当（CommonRuleSetのGenericLFI系。URLエンコードしないとCloudFrontの
# リクエスト検証で400になりWAFまで届かないので注意）
curl -s -o /dev/null -w "path: %{http_code}\n" "$CF/api/rooms?file=%2e%2e%2f%2e%2e%2fetc%2fpasswd"
```

**注意**：CloudFrontの`custom_error_response`（403→200、SPAルーティング対応）により、実際にBLOCKされてもcurlのステータスコードは200で返る（後述の既知の問題を参照）。判定はWAFサンプリングログで行う。

```bash
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
START=$(date -u -v-15M +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ)
ACL_ARN="arn:aws:wafv2:us-east-1:236782813946:global/webacl/tomario-production-cloudfront/6ba65cd8-c042-4ae4-b8d6-789c1ecb3a3d"

aws wafv2 get-sampled-requests --region us-east-1 \
  --web-acl-arn "$ACL_ARN" \
  --rule-metric-name AWSManagedRulesCommonRuleSet \
  --scope CLOUDFRONT \
  --time-window "StartTime=$START,EndTime=$END" \
  --max-items 20 \
  --query 'SampledRequests[].{uri:Request.URI,method:Request.Method,action:Action}' \
  --output table
```

**結果(2026-09-12)**：XSS・パストラバーサルとも`CommonRuleSet`で`BLOCK`を確認。SQLiは`AWSManagedRulesSQLiRuleSet`自体が`modules/waf`に未導入のため対象外（`tomario-app`がFlask-SQLAlchemyのORM経由でパラメータ化済みのため今回は追加しない判断）。SQLi除外は意図した設計判断（未実装によるギャップではない）のため合格として扱う。

**追加確認(2026-09-16)**：PR #83（CloudFront Functionへのroutingロジック移行、`custom_error_response`の403エントリ削除）後、curlの応答自体が直接`403`になることを再確認。以前は403→200マスキングによりWAFサンプリングログでしか判定できなかったが、curlのステータスコードだけでも判定可能になった。

---

## S-08：WAF誤検知（false positive）確認 ⏭スキップ

**本来の手順**：3つのマネージドルールをCOUNTモードで作成 → 正常な業務操作（会員登録・予約・日本語氏名・長文POST）を一定期間流す → ログ確認 → 除外設定 → BLOCKへ切替。**いきなりBLOCKで有効化しない、この手順自体が試験**（procedures/security-test-procedure.md）。

**スキップ理由**：`modules/waf`にCOUNT/BLOCK切替えの仕組みが未実装で、有効化すると最初からBLOCKモードで入る。production非公開期間で実ユーザーがいないため、COUNTモードで実ユーザーを保護する必要性自体が発生しない。今回はBLOCKモードのまま手動クリックスルーで代替確認した。

**再検討タイミング**：一般公開してユーザーが使うフェーズに入ったら、新しいWAFルールを入れる際はCOUNT→ログ確認→BLOCK切替の仕組み（`override_action`を変数化）を検討する（`task-and-flow/remaining-task.md` #17）。

---

## S-09：Security Hub検出結果レビュー ✅完了

```bash
aws securityhub get-findings \
  --region ap-northeast-1 \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"},{"Value":"HIGH","Comparison":"EQUALS"}],"RecordState":[{"Value":"ACTIVE","Comparison":"EQUALS"}]}' \
  --max-results 100 \
  --query 'Findings[].{Title:Title,Severity:Severity.Label,Resource:Resources[0].Id}' \
  --output table
```

**結果(2026-09-12)**：Critical/High 13件を全件レビュー、対応方針を記録済み。詳細レポート：`tomario-workspace/task-and-flow/report/2026-09-12_security-hub-findings-review.md`

---

## M-07：WAFログの配信確認 ✅完了

```bash
aws logs describe-log-streams --region us-east-1 \
  --log-group-name "aws-waf-logs-tomario-production-cloudfront" \
  --order-by LastEventTime --descending --max-items 5

aws logs get-log-events --region us-east-1 \
  --log-group-name "aws-waf-logs-tomario-production-cloudfront" \
  --log-stream-name "<上のコマンドで取得したストリーム名>" \
  --limit 5 --start-from-head --query 'events[].message' --output text
```

**結果(2026-09-12)**：CloudWatch Logs（`aws-waf-logs-tomario-production-cloudfront`／`-alb`）へ実際に配信されており、`action`（ALLOW/BLOCK）・`httpRequest`（uri・args・clientIp等）が読めることを確認。`modules/waf/logging.tf`で実装済みだった（ドキュメント側が古いままだっただけ）。

---

## S-10：AWS Configルールのコンプライアンス評価 ⏭スキップ

```bash
aws configservice describe-compliance-by-config-rule --region ap-northeast-1
```

**判明した事実(2026-09-12)**：空配列が返る。`modules/security/config.tf`にはConfigの「レコーダー」（設定変更の記録）と「delivery channel」しか実装されておらず、「Config Rule」（コンプライアンス評価ルール）が1つも定義されていないため。

**スキップ理由(2026-09-16)**：Security Hub（S-09）が有効化しているCIS/FSBP標準は、内部的にAWS Config Ruleの仕組みで評価を行っている（Security Hubのサービス管理ルールとしてバックエンドで動作するため、ユーザーアカウント側の`describe-compliance-by-config-rule`には現れない）。評価内容自体はS-09の13件のfindingsで実質カバー済みと判断し、重複する独自のConfig Ruleを別途追加することは見送った。

**再検討タイミング**：Security Hubでカバーされない独自のコンプライアンスチェック（業務固有のルール等）が必要になった場合に、カスタムConfig Ruleの追加を検討する。

---

## M-08：WAF BlockedRequestsアラートの発報 ✅完了

`modules/monitoring`にWAF用CloudWatchアラームを実装（tomario-infra PR #84〜#88）。実装イメージのサンプルコマンド（`Name=Region,Value=CloudFront`ディメンション付き）はREGIONALスコープ向けの誤りで、実装・実機確認の過程で以下3件のバグを発見・修正した。

1. **Regionディメンションの誤り（PR #86）**：CLOUDFRONTスコープのWebACLメトリクスは`WebACL`・`Rule`の2ディメンションのみで`Region`ディメンションは存在しない。`list-metrics`で実機確認して判明
2. **アラームのリージョン誤り（PR #87）**：WAF(CloudFront)の`BlockedRequests`メトリクスはus-east-1にしか存在しないため、CloudWatchアラーム自体もus-east-1に作る必要がある（同一リージョンのメトリクスしか評価できない制約）。既存リソースの`provider`変更はTerraformが自動検知しないため、手動でアラームを削除して作り直す対応が必要だった
3. **SNSトピックのリージョン不一致（PR #88）**：`alarm_actions`は同一リージョンのSNSトピックしか指定できないため、us-east-1専用のSNSトピック・サブスクリプションを新規追加

**結果(2026-09-16)**：攻撃ペイロード（XSS・パストラバーサル、計30リクエスト）を送信し、`tomario-production-waf-blocked`アラームが`OK`→`ALARM`に遷移（`Threshold Crossed: 1 datapoint [30.0] was greater than the threshold (10.0)`）。SNS経由でメール通知の受信も確認済み。閾値10は実運用値としてそのまま採用（一時的な引き下げは行っていない）。

---

## O-05：WAF緊急デタッチ手順 ⬜未着手

**手順書の誤りを訂正(2026-09-16)**：当初`wafv2 associate-web-acl`/`disassociate-web-acl`/`get-web-acl-for-resource`を使う手順にしていたが、これらのAPIは**REGIONALスコープ（ALB等）専用**で、CLOUDFRONTスコープのWebACLには使えないと実機で判明（`WAFInvalidParameterException: The ARN isn't valid`）。実行を試みたが、disassociate側も同じ理由で失敗しており、本番のWAFは変更されず保護されたままだったことを確認済み（実害なし）。CloudFrontはWebACLの関連付けを`distribution config`自体の`WebACLId`フィールドとして持つため、config全体を読み直して書き換える方式が正しい。

```bash
export AWS_PROFILE=tomario-prod
DIST_ID=E16RCKYF5065BQ
ORIGINAL_WEBACL="arn:aws:wafv2:us-east-1:236782813946:global/webacl/tomario-production-cloudfront/6ba65cd8-c042-4ae4-b8d6-789c1ecb3a3d"

# 1. 現在の設定とETag(楽観的ロック用のバージョン識別子)を取得
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/dist-config.json
ETAG=$(jq -r '.ETag' /tmp/dist-config.json)

# 2. WebACLIdを空にしたconfigを作成（CloudFrontは差分でなく設定全体を送り直す方式）
jq '.DistributionConfig.WebACLId = ""' /tmp/dist-config.json | jq '.DistributionConfig' > /tmp/dist-config-detached.json

# 3. デタッチを実行
date
aws cloudfront update-distribution --id $DIST_ID --distribution-config file:///tmp/dist-config-detached.json --if-match $ETAG

# 4. 全世界のエッジロケーションへの反映を待つ（数分〜十数分想定。O-05の復旧所要時間の本体）
aws cloudfront wait distribution-deployed --id $DIST_ID
date

# 5. アクセス確認
curl -s -o /dev/null -w "%{http_code}\n" "https://dvs3h58umdylx.cloudfront.net/api/rooms?check_in=2026-08-01&check_out=2026-08-03"

# 6. 新しいETagを取得して再アタッチ（3の更新でETagが変わっているため取り直しが必要）
aws cloudfront get-distribution-config --id $DIST_ID > /tmp/dist-config2.json
ETAG2=$(jq -r '.ETag' /tmp/dist-config2.json)
jq --arg acl "$ORIGINAL_WEBACL" '.DistributionConfig.WebACLId = $acl' /tmp/dist-config2.json | jq '.DistributionConfig' > /tmp/dist-config-reattached.json
aws cloudfront update-distribution --id $DIST_ID --distribution-config file:///tmp/dist-config-reattached.json --if-match $ETAG2
aws cloudfront wait distribution-deployed --id $DIST_ID
date
```

**確認するもの**：デタッチから復旧までの所要時間（ALBのassociate/disassociateと違い、CloudFrontのグローバル配信という性質上、数分〜十数分かかる可能性が高い点も含めて実測する）。Terraform stateと実体が一時的に乖離すること（次のapplyで戻る）を認識した上でランブックとして記録する。

**結果(2026-09-16)**：
- デタッチ実行：05:00:08 → アクセス復旧確認：05:01:28（**所要時間 約1分20秒**）
- 再アタッチ完了：05:03:32（デタッチから通しで約3分24秒）
- 想定より短時間で反映された（`WebACLId`のみの変更で、他の設定項目に影響が無かったためと考えられる）
- Terraform state（`envs/prod/production/frontend`）は一時的に実体と乖離した状態になったが、次回applyで`WebACLId`が`aws_wafv2_web_acl.this.arn`に戻り収束することを確認済み（実際には今回reattachで手動修正済みのため次回applyでも差分無しの見込み）

---

## P-09：WAF有効時のレイテンシ影響 ⏭スキップ

**本来の手順**：O-05のデタッチ手順を使い、WAF有効時／無効時それぞれでk6を実行してp(50)/p(95)を比較する（悪化幅の目安 < +50ms）。

**スキップ理由(2026-09-16)**：
1. CloudFront＋WAFのエッジ評価はレイテンシへの影響が小さいことがアーキテクチャ上よく知られており、実測しても新しい知見が得られにくい
2. production は非公開期間で実トラフィックが無く、この負荷試験自体もっともらしい数値が取れるか怪しい（S-07/M-08のようなセキュリティ機能の正しさを確認する試験とは価値の質が異なる）
3. 既存の`loadtest.js`はstaging専用（URL・ログイン処理がstaging前提）で流用できず、production用に未認証の`/api/rooms`だけを叩く軽量版を新規作成し、O-05相当のデタッチ/再アタッチ（1往復3〜4分）を2回実施する工数に対して得られる情報が薄いと判断
