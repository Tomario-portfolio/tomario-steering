# 非機能試験（スケーラビリティ）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の性能試験（P-01〜P-03）
対応する結果報告書：[../results/performance-test-result.md](../results/performance-test-result.md)

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
| 監視 | ターミナルを 2 つ使用。別のターミナルで監視を起動してから、ターミナルで k6 を開始する |
| RDS | 本格的な負荷時のみ db.t4g.medium へ一時スケールアップし、**試験後に必ず db.t3.micro へ戻す**（詳細手順4） |
| Auto Scaling 設定 | min=2 / max=4 / target CPU 70%、スケールイン側はクールダウン 5 分 + 15 分連続閾値未満 |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| P-01 | 負荷時のスケーリングと可用性 | staging | 負荷をかけ、スケールアウト・スケールイン、成功率と応答時間を確認する（詳細手順2を参照） | desiredCount が 2 → 3 → 4 へ段階的に増加し、スケーリングアクティビティが全件 `Successful`。負荷終了後 15 分以内に min(2) へ戻る。成功率 ≥ 99%。p(95) は実測値を記録 | ログインは PBKDF2 照合で CPU コストが高い。GET だけだと CPU が上がりきらない |
| P-02 | 目標スループットの確認 | staging | 目標 rps で負荷をかけ、成功率と応答時間を確認する（詳細手順3を参照） | 目標 rps を維持した状態で成功率 ≥ 99%、p(95) が許容内 | **目標値（目標 rps・p95 許容）は非機能要件定義書へ追記後に確定**。現状は暫定値で実測し、要件にフィードバックする |
| P-03 | WAF によるレイテンシ影響 | production | WAF 有効・無効で同じ負荷をかけ、応答時間を比較する（詳細手順5を参照） | WAF 有効時の p50 / p95 の悪化幅が小さい（目安 < +50ms） | BLOCK 切替後に実施 |

## 詳細手順

### 実施順序

```
手順 1（準備：ベースライン測定・RDS スケールアップ） → 手順 2（P-01） → 手順 3（P-02） → 手順 4（後始末：RDS を戻す）
手順 5（P-03）は production で別に実施する
```

### 手順 1：準備（ベースライン測定・RDS スケールアップ）

#### 1-1. ベースライン測定

負荷なしの状態で API の応答時間を数回測り、ECS の CPU / メモリ使用率を記録する。P-01・P-02 の評価の基準にする。

```bash
# API 応答時間を 10 回計測
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{time_total}\n" "https://{CLOUDFRONT_DOMAIN}/api/rooms?check_in=2026-08-01&check_out=2026-08-03"
done

# ECS の CPU 使用率（直近 30 分、5 分平均）。MemoryUtilization も同様に取得する
aws cloudwatch get-metric-statistics --namespace AWS/ECS --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=tomario-staging-cluster Name=ServiceName,Value=tomario-staging-service \
  --start-time $(date -u -v-30M +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

#### 1-2. RDS 一時スケールアップ

RDS がボトルネックにならないよう、一時的に db.t4g.medium へ変更する。

```bash
aws rds modify-db-instance --db-instance-identifier tomario-staging-rds \
  --db-instance-class db.t4g.medium --apply-immediately
aws rds wait db-instance-available --db-instance-identifier tomario-staging-rds
aws rds describe-db-instances --db-instance-identifier tomario-staging-rds \
  --query "DBInstances[0].DBInstanceClass"
```

### 手順 2（P-01）：負荷時のスケーリングと可用性

1. 別のターミナルで、サービスのタスク数を 10 秒間隔で監視する
2. ターミナルで k6 の負荷シナリオ（ログイン → 部屋一覧、50 VU / 計 9 分）を実行する
3. 負荷中にタスク数が 2 → 3 → 4 と増えることを確認し、終了後にスケーリングアクティビティがすべて成功していることを確認する
4. k6 のサマリから成功率と p(95) を記録し、ベースラインと比較する
5. k6 の終了後 10〜15 分、タスク数が 2 に戻ることを確認する
6. ダッシュボード `tomario-staging-autoscaling` でタスク数の推移（2 → 4 → 2）を確認し、スクリーンショットを証跡として残す

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

# 【別のターミナル】k6 開始前から監視を起動し、スケールインまで観察する
watch -n 10 'aws ecs describe-services --cluster tomario-staging-cluster --services tomario-staging-service \
  --query "services[0].{desired:desiredCount,running:runningCount}"'

# 【ターミナル】負荷を実行
k6 run loadtest.js

# k6 終了後、スケーリングアクティビティを確認（Failed なら IAM 権限不足等。Description を確認）
aws application-autoscaling describe-scaling-activities --service-namespace ecs \
  --resource-id service/tomario-staging-cluster/tomario-staging-service \
  --query "ScalingActivities[*].{Time:StartTime,Description:Description,Status:StatusCode}" --output table
```

### 手順 3（P-02）：目標スループットの確認

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

### 手順 4：後始末（RDS を元に戻す）

忘れると割高なクラスの課金が続く。次回の `terraform apply` でも戻るが、それまで課金される。

```bash
aws rds modify-db-instance --db-instance-identifier tomario-staging-rds \
  --db-instance-class db.t3.micro --apply-immediately
aws rds wait db-instance-available --db-instance-identifier tomario-staging-rds
```

### 手順 5（P-03）：WAF によるレイテンシ影響（production）

WAF の付け外しは、運用の手順書（[operations-test-procedure.md](operations-test-procedure.md)）の詳細手順 2（O-02）と同じ方法で行う。

```bash
# 1) WAF を CloudFront に関連付けた状態で loadtest.js を実行 → k6 サマリの p(50)/p(95) を記録
k6 run loadtest.js   # BASE_URL は production の CloudFront

# 2) WAF の関連付けを外す（O-02 詳細手順の 1〜2）

# 3) 同じ loadtest.js をもう一度実行 → p(50)/p(95) を記録
k6 run loadtest.js

# 4) WAF を関連付け直す（O-02 詳細手順の 4）
```

確認するもの：WAF 有効 / 無効での p(50)・p(95) の差分。悪化幅が目安（< +50ms）を超える場合はルール構成を見直す。

## 実施後の記録

- 結果を [../results/performance-test-result.md](../results/performance-test-result.md) の結果表へ転記する
- p(95) 実測値と P-02 の実効 rps をもとに、非機能要件定義書へレスポンスタイム目標値（例：定常時 p95 < 500ms）と目標 rps を追記し、`test-plan.md` の P-02 目標値を確定する
- `loadtest.js`・`loadtest-throughput.js`・k6 サマリ・スケーリングアクティビティ一覧を `../evidence/performance/` に格納する
- **後始末（RDS を db.t3.micro へ戻した）を必ずチェックする**
