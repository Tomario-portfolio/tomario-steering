# 非機能試験（セキュリティ）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) のセキュリティ試験（S-01〜S-10）
対応する結果報告書：[../results/security-test-result.md](../results/security-test-result.md)

## 試験実施目的

セキュリティリソースを「有効化した」ことと、「実際に脅威をブロックし、証跡を残し、既知の脆弱性がない」
ことは別問題である。コードを読んで把握した設計が、実環境で意図どおり機能しているかを確認する。

- アプリ依存パッケージ・コンテナイメージに既知の重大な脆弱性がないこと
- 通信が TLS 1.2 以上に限定され、弱い暗号スイートが無効なこと
- ネットワーク境界（ALB 直アクセス拒否・RDS 外部到達不可・ECS パブリック IP なし・S3 非公開）が意図どおりであること
- 脅威検知（GuardDuty）と監査証跡（CloudTrail）が実際に機能していること

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | S-01/S-02 は CI と ECR、S-03 は production の CloudFront、S-04/S-05 は各環境、S-07〜S-10 は production（未公開期間） |
| 未整備事項 | `pip-audit` は `tomario-app` への導入自体が未対応（remaining-task SEC-4）。ECR プッシュ時スキャンは有効だが結果レビュー運用が未整備 |
| WAF 前提（S-07/S-08） | `security-environment-design.md` の方針：CloudFront 用（および ALB 用）Web ACL に `AWSManagedRulesCommonRuleSet` / `AWSManagedRulesKnownBadInputsRuleSet` / `AWSManagedRulesSQLiRuleSet` の 3 つをアタッチ。**WAF ログの配信先（S3 / CloudWatch Logs / Firehose）が環境定義に未記載＝先に決めて実装が必要**。IAM は bootstrap のインフラ用ポリシーに `wafv2` 権限が必要（[[feedback_iam_permission_check]]） |
| Security Hub / Config 前提（S-09/S-10） | `enable_security_hub` / `enable_config` を production で `true` にして apply。IAM に `securityhub` / `config` 権限が必要 |
| 稼働環境スキャン時の注意 | AWS 上の稼働環境に対して能動スキャンを行う場合は [AWS Customer Support Policy for Penetration Testing](https://aws.amazon.com/security/penetration-testing/) を確認し、禁止行為（DDoS/DoS シミュレーション、ポート/プロトコルフラッディング等）に該当しないことを確認してから実施する |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| S-01 | 依存パッケージの脆弱性スキャン | CI（tomario-app） | `deploy.yml` に `pip-audit`（または `pip-audit -r requirements.txt`）ステップを追加し、Critical/High 検出でジョブを失敗させる。ローカル先行実行も可 | Critical/High ゼロ、または各指摘に対応方針（バージョン更新／許容理由）を記録 | SEC-4。まず tomario-app への導入が必要 |
| S-02 | コンテナイメージの脆弱性スキャン | ECR / ローカル | ECR プッシュ時スキャン結果を `aws ecr describe-image-scan-findings --repository-name tomario-production-app --image-id imageTag=<tag>` で取得。または `trivy image <ECR image URI>` | Critical/High ゼロ、または対応方針を記録 | ベースイメージ更新で解消するものが多い |
| S-03 | TLS 設定の確認 | production | `testssl.sh <CloudFront ドメイン>` または SSL Labs（`https://www.ssllabs.com/ssltest/`）で診断 | TLS 1.2 以上のみ有効。既知の弱いプロトコル（SSLv3/TLS1.0/1.1）・弱い暗号スイートが無効 | CloudFront のセキュリティポリシー設定に依存 |
| S-04 | ネットワーク境界の構成確認 | dev / staging / production | 詳細手順 1。(a) ALB 直アクセスが 403（X-Origin-Verify, SEC-7） (b) ローカルから RDS:3306 がタイムアウト (c) ECS タスクにパブリック IP なし (d) S3 の Block Public Access が有効 | すべて想定どおり（拒否・到達不可・IP なし・公開ブロック） | (a)(c) は既に実施実績あり。項番付きで再確認 |
| S-05 | 脅威検知・監査証跡の有効性 | dev / staging / production | 詳細手順 2。GuardDuty が `ENABLED`、CloudTrail に直近の操作（RunTask / GetSecretValue 等）が記録され、S3 バケットにログファイルが存在する | GuardDuty 有効、CloudTrail のイベントと S3 保存を確認 | 「有効化した」と「記録されている」は別 |
| S-07 | WAF マネージドルールの有効性 | production（未公開期間） | 詳細手順 3。SQLi・XSS・パストラバーサル等の攻撃ペイロードを curl / OWASP ZAP で送信し、403（WAF ブロック）が返ることを確認。`CommonRuleSet` / `KnownBadInputsRuleSet` / `SQLiRuleSet` の各カテゴリで実施 | 攻撃リクエストが 403 でブロックされ、WAF ログに `action=BLOCK` が記録される | 「WAF を入れた」と「効いている」は別 |
| S-08 | WAF 誤検知（false positive）確認 | production（未公開期間） | 詳細手順 4。3 つのマネージドルールをまず COUNT モードで作成 → 数日〜一定期間、正常な業務操作（会員登録・予約フォーム送信・日本語氏名・長文 POST）を流す → WAF ログのサンプルを確認 → 誤検知ルールを除外（rule action override / label match）→ BLOCK へ切替 | 正常操作が COUNT ログで BLOCK 相当に該当しない、または除外設定後に該当しなくなる | **いきなり BLOCK で有効化しない**。この手順自体が試験 |
| S-09 | Security Hub の検出結果レビュー | production（未公開期間） | `enable_security_hub=true` で apply 後、`aws securityhub get-findings --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"},{"Value":"HIGH","Comparison":"EQUALS"}]}'` で Critical / High を抽出しレビュー | Critical / High = 0、または各 finding に「対応」「許容（理由）」を記録 | CIS AWS Foundations 等のセキュリティ標準に対する自動評価 |
| S-10 | AWS Config ルールのコンプライアンス評価 | production（未公開期間） | `enable_config=true` で apply 後、`aws configservice describe-compliance-by-config-rule` / `get-compliance-details-by-config-rule` で NON_COMPLIANT を抽出しレビュー | NON_COMPLIANT = 0、または各リソースに対応方針を記録 | リソース構成のコンプライアンス継続監視 |
| S-06 | IAM 最小権限の棚卸し（任意） | 全環境 | IAM Access Analyzer の未使用アクセス分析、または `aws iam generate-service-last-accessed-details` で各ロールの未使用権限を確認 | 未使用の広範な権限がない、または削減方針を記録 | 優先度低。職務分掌（インフラ用／デプロイ用）は設計済み |

## 詳細手順

### 手順 1（S-04）：ネットワーク境界

```bash
# (a) ALB 直アクセスが 403（X-Origin-Verify ヘッダーなし）
ALB_DNS=$(aws elbv2 describe-load-balancers --names tomario-staging-alb \
  --query "LoadBalancers[0].DNSName" --output text)
curl -s -o /dev/null -w "%{http_code}\n" "http://$ALB_DNS/api/rooms"      # → 403 を期待

# (b) ローカル PC から RDS:3306 がタイムアウト／拒否
RDS_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier tomario-staging-rds \
  --query "DBInstances[0].Endpoint.Address" --output text)
nc -zv "$RDS_ENDPOINT" 3306 -w 5                                          # → timed out / refused を期待

# (c) ECS タスクにパブリック IP が付いていない
TASK_ARN=$(aws ecs list-tasks --cluster tomario-staging-cluster --query "taskArns[0]" --output text)
ENI=$(aws ecs describe-tasks --cluster tomario-staging-cluster --tasks $TASK_ARN \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
aws ec2 describe-network-interfaces --network-interface-ids $ENI \
  --query "NetworkInterfaces[0].Association.PublicIp" --output text        # → None / 空 を期待

# (d) S3 の Block Public Access
for B in tomario-staging-logs-$(aws sts get-caller-identity --query Account --output text); do
  aws s3api get-public-access-block --bucket "$B" \
    --query "PublicAccessBlockConfiguration"                              # → 4 項目すべて true を期待
done
```

### 手順 2（S-05）：脅威検知・監査証跡

```bash
# GuardDuty
aws guardduty list-detectors --query "DetectorIds" --output text | \
  xargs -I{} aws guardduty get-detector --detector-id {} --query "{Status:Status,UpdatedAt:UpdatedAt}"
#   → "Status": "ENABLED"

# CloudTrail：直近 1 時間の RunTask イベント
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunTask \
  --start-time $(date -v-1H +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d "1 hour ago" +%Y-%m-%dT%H:%M:%S) \
  --query "Events[*].{Time:EventTime,Name:EventName,User:Username}" --output table

# CloudTrail：S3 への長期保存
aws s3 ls s3://tomario-shared-cloudtrail-$(aws sts get-caller-identity --query Account --output text)/ --recursive | head
#   → AWSLogs/<account>/CloudTrail/ 配下に JSON が存在
```

### 手順 3（S-07）：WAF マネージドルールの有効性

前提：WAF が BLOCK モードで CloudFront に関連付いていること（S-08 の切替後）。`<CF>` は production の CloudFront ドメイン。

```bash
CF="https://<CLOUDFRONT_DOMAIN>"

# 正常リクエスト（ブロックされないベースライン）
curl -s -o /dev/null -w "normal: %{http_code}\n" "$CF/api/rooms?check_in=2026-08-01&check_out=2026-08-03"

# SQLi 相当（SQLiRuleSet / CommonRuleSet）
curl -s -o /dev/null -w "sqli:   %{http_code}\n" "$CF/api/rooms?check_in=2026-08-01%27%20OR%20%271%27=%271"

# XSS 相当（CommonRuleSet）
curl -s -o /dev/null -w "xss:    %{http_code}\n" --data-urlencode "q=<script>alert(1)</script>" "$CF/api/rooms"

# KnownBadInputs 相当（パストラバーサル等）
curl -s -o /dev/null -w "path:   %{http_code}\n" "$CF/../../etc/passwd"
```

確認するもの：攻撃系が `403`（WAF ブロック）、正常系は `200`。WAF ログ（M-07 の配信先）に該当リクエストが `action=BLOCK` / `terminatingRuleId` 付きで残ること。

### 手順 4（S-08）：WAF 誤検知（false positive）確認

```
1. Web ACL を作成し、3 つのマネージドルールを OverrideAction=Count（COUNT モード）でアタッチ
   （BLOCK にしない。ログだけ取る）
2. CloudFront へ関連付け
3. 一定期間、正常な業務操作を実際に流す：
   - 会員登録（日本語氏名・記号入りパスワード）
   - 予約作成（長文の要望欄など、可能なら）
   - 部屋検索の各種クエリ
   （k6 の loadtest.js を短時間流すのも可）
4. WAF ログを確認し、正常リクエストが COUNT（＝BLOCK 相当）に該当していないか調べる
```

```bash
# COUNT に該当したリクエストの terminatingRuleId / ラベルを集計（配信先が CloudWatch Logs の場合）
aws logs start-query \
  --log-group-name "<WAF ロググループ>" \
  --start-time $(date -v-1d +%s 2>/dev/null || date -d '1 day ago' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, httpRequest.uri, terminatingRuleId, action
                  | filter action = "COUNT" or action = "BLOCK"
                  | stats count() by terminatingRuleId, httpRequest.uri
                  | sort count() desc | limit 50'
aws logs get-query-results --query-id <queryId>
```

対応：誤検知しているルールは `RuleActionOverrides`（特定ルールだけ Count のまま）や `ScopeDownStatement` / label match で除外設定を追加する。
誤検知が無くなったことを確認してから、OverrideAction を外して BLOCK へ切り替える。

## 実施後の記録

- 結果を [../results/security-test-result.md](../results/security-test-result.md) に転記し、ステータスを「実施済み」に更新する
- 検出された指摘は 1 件ずつ「対応（更新）」「許容（理由）」を記録する
- スキャンレポート・testssl.sh 出力・各確認コマンドの結果を `../evidence/security/` に格納する（アカウント ID・エンドポイント等はマスク）
