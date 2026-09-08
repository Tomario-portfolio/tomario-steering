# 非機能試験（スケーラビリティ）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の性能試験（P-00・P-03〜P-05・P-08・P-09）
対応する結果報告書：[../results/scalability-test-result.md](../results/scalability-test-result.md)

## 試験実施目的

「Application Auto Scaling を設定した」ことと「実際に閾値どおりスケールする」ことは別問題である。
target tracking の設定ミスや、CPU 使用率が閾値に届かない負荷では検証にならない。実負荷をかけて確認する。

- 高負荷時に desiredCount が min(2) から max(4) へ段階的に増加すること
- 負荷終了後に min(2) までスケールインすること
- 高負荷時もリクエストの可用性（成功率）を維持すること
- 負荷なし時のベースライン値を取得し、比較の基準にすること
- 目標スループット（目標 rps）のリクエストを、判定基準を満たしつつ捌けること

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging（`tomario-staging-cluster` / `tomario-staging-service`） |
| テストデータ | users 2,000 件 / bookings 5,000 件を投入済み（空 DB への負荷は非現実的） |
| ツール | k6 インストール済み（`brew install k6`） |
| 監視 | ターミナルを 2 つ使用。B で監視を起動してから A で k6 を開始する |
| RDS | 本格的な負荷時のみ db.t4g.medium へ一時スケールアップし、**試験後に必ず db.t3.micro へ戻す**（番号 P-06） |
| Auto Scaling 設定 | min=2 / max=4 / target CPU 70%、スケールイン側はクールダウン 5 分 + 15 分連続閾値未満 |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| P-00 | ベースライン測定 | staging | 負荷なしの状態で、API 応答時間（`curl -w "%{time_total}"` を数回）と ECS の CPU / メモリ使用率（`aws cloudwatch get-metric-statistics`）を記録 | 定常状態の基準値を取得 | 比較の基準。P-05・P-08 の評価に使う |
| P-01 | RDS 一時スケールアップ | staging | `aws rds modify-db-instance --db-instance-identifier tomario-staging-rds --db-instance-class db.t4g.medium --apply-immediately` → `wait db-instance-available` → `describe-db-instances --query "DBInstances[0].DBInstanceClass"` | `db.t4g.medium` になる | RDS がボトルネックにならないようにするため |
| P-02 | k6 で負荷投入 | staging | 詳細手順 1。`loadtest.js`（ログイン → 部屋一覧、50 VU / 計 9 分）を作成し `k6 run loadtest.js` | k6 サマリの checks がほぼ 100%、`http_req_duration` を記録 | ログインは PBKDF2 照合で CPU コストが高い。GET だけだと CPU が上がりきらない |
| P-03 | スケールアウト確認 | staging | 【ターミナル B】`watch -n 10 'aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service --query "services[0].{desired:desiredCount,running:runningCount}"'` を k6 開始前から起動。負荷中の推移を観察し、後で `aws application-autoscaling describe-scaling-activities --service-namespace ecs --resource-id service/tomario-staging-cluster/tomario-staging-service --query "ScalingActivities[*].{Time:StartTime,Description:Description,Status:StatusCode}" --output table` | desiredCount が 2 → 3 → 4 へ段階的に増加。スケーリングアクティビティが全件 `Successful` | `Failed` は IAM 権限不足等。Description を確認 |
| P-04 | スケールイン確認 | staging | k6 終了後 10〜15 分、同じ watch コマンドで監視 | desiredCount が 15 分以内に min(2) へ戻る | スケールアウト（約 3 分）より時間がかかる |
| P-05 | 可用性・レスポンスタイム評価 | staging | k6 サマリの `checks_succeeded`（成功率）と `http_req_duration` の p(95) を P-00 のベースラインと比較 | 成功率 ≥ 99%。p(95) は実測値を記録（目標値は非機能要件へフィードバック） | P-03（可用性）・P-04（参考）に対応 |
| P-06 | RDS を元に戻す | staging | `aws rds modify-db-instance --db-instance-identifier tomario-staging-rds --db-instance-class db.t3.micro --apply-immediately` → `wait db-instance-available` | `db.t3.micro` に復帰 | 忘れると割高なクラスの課金が継続。次回 `terraform apply` でも戻るがそれまで課金 |
| P-07 | ダッシュボード可視化（任意） | staging | `tomario-staging-autoscaling` ダッシュボードで負荷試験時間帯のタスク数推移を確認・スクショ | 山型（2→4→2）のグラフ | RunningTaskCount は Container Insights 未有効のため現状未表示（既知課題） |
| P-08 | 目標スループット達成確認 | staging | 詳細手順 2。非機能要件に定めた目標 rps を狙って k6 の `stages` を調整（例：目標 rps に対応する VU 数で 5〜10 分維持）。`http_reqs` の rps・`checks_succeeded`・`http_req_duration` p(95) を取得 | 目標 rps を維持した状態で 成功率 ≥ 99%、p(95) が許容内 | **目標値（目標 rps・p95 許容）は非機能要件定義書へ追記後に確定**。現状は暫定値で実測し、要件にフィードバックする |
| P-09 | WAF 有効時のレイテンシ影響 | production（未公開期間） | 詳細手順 3。`loadtest.js` 相当の負荷を、WAF を CloudFront に関連付けた状態と外した状態の両方で流し、k6 サマリの `http_req_duration` の p(50) / p(95) を比較 | WAF 有効時の p50 / p95 の悪化幅が小さい（目安 < +50ms） | WAF は通常オーバーヘッド小さいが「測った」と言えるようにする。BLOCK 切替後に実施 |

## 詳細手順

### 手順 1（P-02）：k6 シナリオ作成と実行

```bash
# CloudFront ドメインを確認
aws cloudfront list-distributions \
  --query "DistributionList.Items[*].{Domain:DomainName,Comment:Comment}" --output table

cat > loadtest.js << 'EOF'
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 50 },
    { duration: '5m', target: 50 },
    { duration: '2m', target: 0 },
  ],
};

const BASE_URL = 'https://{CLOUDFRONT_DOMAIN}';   // ← 実値に置換

export default function () {
  const userId = Math.floor(Math.random() * 2000);
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: `loadtest_user_${userId}@example.com`,
    password: 'loadtest-password-not-for-prod',
  }), { headers: { 'Content-Type': 'application/json' } });
  check(loginRes, { 'login status is 200': (r) => r.status === 200 });

  const roomsRes = http.get(`${BASE_URL}/api/rooms?check_in=2026-08-01&check_out=2026-08-03`, {
    cookies: loginRes.cookies,
  });
  check(roomsRes, { 'rooms status is 200': (r) => r.status === 200 });
  sleep(0.5);
}
EOF

# 【ターミナル B】で監視を起動してから、【ターミナル A】で実行
k6 run loadtest.js
```

### 手順 2（P-08）：目標スループット達成確認

前提：非機能要件定義書に目標 rps（および p95 許容値）が定義されていること。未定義なら暫定値を置いて実測し、結果を要件へフィードバックする。

```bash
# 目標 rps に合わせて stages を調整。目安：目標 rps ÷ (1 / 1リクエストあたりの平均所要秒) ≒ 必要 VU
# 例）目標 100 rps、1 リクエスト平均 0.25s + sleep 0.5s → 1 VU ≒ 1.3 rps → 約 75〜80 VU
cat > loadtest-throughput.js << 'EOF'
import http from 'k6/http';
import { sleep, check } from 'k6';
export const options = {
  stages: [
    { duration: '2m', target: 80 },
    { duration: '8m', target: 80 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_failed: ['rate<0.01'],   // 失敗率 1% 未満
    // http_req_duration: ['p(95)<800'], // ← 要件確定後に有効化
  },
};
const BASE_URL = 'https://{CLOUDFRONT_DOMAIN}';
export default function () {
  const userId = Math.floor(Math.random() * 2000);
  const login = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: `loadtest_user_${userId}@example.com`, password: 'loadtest-password-not-for-prod',
  }), { headers: { 'Content-Type': 'application/json' } });
  check(login, { 'login 200': (r) => r.status === 200 });
  const rooms = http.get(`${BASE_URL}/api/rooms?check_in=2026-08-01&check_out=2026-08-03`, { cookies: login.cookies });
  check(rooms, { 'rooms 200': (r) => r.status === 200 });
  sleep(0.5);
}
EOF

k6 run loadtest-throughput.js
# サマリから http_reqs の rps、checks_succeeded、http_req_duration p(95) を記録
```

確認するもの：維持できた実効 rps、成功率 ≥ 99%、p(95) の実測値。目標未達ならボトルネック（ECS CPU / RDS 接続 / CloudFront）を考察する。

### 手順 3（P-09）：WAF 有効時のレイテンシ影響（production）

```bash
# 1) WAF を CloudFront に関連付けた状態で loadtest.js を実行 → k6 サマリの p(50)/p(95) を記録
k6 run loadtest.js   # BASE_URL は production の CloudFront

# 2) WAF の関連付けを一時的に外す（O-05 と同じ手順）
WEBACL_ARN=$(aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1 \
  --query "WebACLs[?Name=='<web acl name>'].ARN" --output text)
DIST_ID=<CloudFront Distribution ID>
aws wafv2 disassociate-web-acl --resource-arn arn:aws:cloudfront::<account>:distribution/$DIST_ID --region us-east-1
# CloudFront への反映を待つ（数分）

# 3) 同じ loadtest.js をもう一度実行 → p(50)/p(95) を記録
k6 run loadtest.js

# 4) WAF を関連付け直す
aws wafv2 associate-web-acl --web-acl-arn $WEBACL_ARN \
  --resource-arn arn:aws:cloudfront::<account>:distribution/$DIST_ID --region us-east-1
```

確認するもの：WAF 有効 / 無効での p(50)・p(95) の差分。悪化幅が目安（< +50ms）を超える場合はルール構成を見直す。

### 実施順序

```
P-00（ベースライン） → P-01（RDS アップ） → P-03 の監視起動 → P-02（k6） →
P-03/P-04 観察 → P-05 評価 → P-08（目標スループット） → P-06（RDS を戻す） → P-07（任意）
```

## 実施後の記録

- 結果を [../results/scalability-test-result.md](../results/scalability-test-result.md) の結果表へ転記する
- p(95) 実測値と P-08 の実効 rps をもとに、非機能要件定義書へレスポンスタイム目標値（例：定常時 p95 < 500ms）と目標 rps を追記し、`test-plan.md` の P-08 目標値を確定する
- `loadtest.js`・`loadtest-throughput.js`・k6 サマリ・スケーリングアクティビティ一覧を `../evidence/scalability/` に格納する
- **P-06（RDS を db.t3.micro へ戻した）を必ずチェックする**
