# 非機能試験（バックアップ）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の復旧試験（B-01・B-03、参考 B-03a）
対応する結果報告書：[../results/backup-test-result.md](../results/backup-test-result.md)

## 試験実施目的

`backup_retention_period = 7` を設定していることと、実際にリストアできることは別問題である。
バックアップ設定だけ入れてリストア手順を一度も試したことがない状態は、実務でよくある事故要因。
以下を実測で確認する。

- 自動バックアップ（continuous backup）が実際に取得され、最新に追従していること
- 誤削除したデータを、削除前の時点へのポイントインタイムリストアで復旧できること
- 復旧に要する時間（RTO）が許容範囲であること

## 前提条件

| 項目 | 内容 |
|---|---|
| 環境 | staging（`tomario-staging-rds`）。手順内のリソース名は staging を例に記載 |
| 起動状態 | cost-start 実行済みで RDS が `available`、ECS サービスが稼働中 |
| コンテナ接続 | ECS Exec が使えること（`ssmmessages` VPC エンドポイントが存在すること）。DB ドライバ `pymysql` はアプリコンテナに同梱済み |
| 認証情報 | RDS マスターパスワードは Secrets Manager から取得（`aws secretsmanager get-secret-value`） |
| 注意 | ポイントインタイムリストアは新規インスタンスとして復元される。訓練後は必ず削除する（番号 B-05） |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| B-01 | 自動バックアップの取得確認 | staging / production | `aws rds describe-db-instances --db-instance-identifier tomario-staging-rds --query "DBInstances[0].{BackupRetention:BackupRetentionPeriod,LatestRestorableTime:LatestRestorableTime,BackupWindow:PreferredBackupWindow}"` | `BackupRetention` が 7、`LatestRestorableTime` が現在時刻の数分〜十数分前 | 古い日時で止まっていたら取得できていないサイン |
| B-02 | 障害シミュレーション（データ削除） | staging | ECS Exec でコンテナに入り、`bookings` の最新レコード 1 件を削除。削除時刻を `date` でメモ | 1 件削除され、削除前の最新 ID を記録できる | 詳細手順 2。壊れた後の時点を復元ポイントにしないための基準時刻 |
| B-03 | ポイントインタイムリストア | staging | 削除時刻の 1 分前を指定して `restore-db-instance-to-point-in-time` → `wait db-instance-available`。開始／完了時刻から RTO を算出 | 新規インスタンス `tomario-staging-rds-restore-test` が作成され、`available` になる。RTO < 30 分 | 詳細手順 3。初回は `LatestRestorableTime` 未到達で `InvalidParameterValue` になることがある（数分待って再実行）。RTO を算出して記録する |
| B-04 | 復元データの検証 | staging | 復元先エンドポイントに接続し `SELECT COUNT(*)` と最新 ID を確認 | B-02 で削除したレコードが復元先に存在する | 詳細手順 4 |
| B-05 | 訓練用インスタンスの削除 | staging | `aws rds delete-db-instance --db-instance-identifier tomario-staging-rds-restore-test --skip-final-snapshot` → 数分後 `describe-db-instances` で `DBInstanceNotFound` | 訓練用インスタンスが削除される | 放置するとストレージ課金が継続するため必須 |
| B-06 | 手動スナップショットからの復元（任意） | staging | `create-db-snapshot` → `restore-db-instance-from-db-snapshot` → 検証 → 復元先削除 | スナップショット取得時点の状態で復元できる | PITR とは別経路の復旧手段の確認。優先度低 |

## 詳細手順

### 手順 2（B-02）：障害シミュレーション

```bash
# コンテナに入る
TASK_ARN=$(aws ecs list-tasks --cluster tomario-staging-cluster \
  --service-name tomario-staging-service --desired-status RUNNING \
  --query 'taskArns[0]' --output text)
aws ecs execute-command --cluster tomario-staging-cluster --task $TASK_ARN \
  --container tomario-app --interactive --command "/bin/sh"

# （コンテナ内）1 件削除。{RDSエンドポイント} {DB_PASSWORD} は実値に置換
python3 -c "
import pymysql
conn = pymysql.connect(host='{RDSエンドポイント}', user='admin', password='{DB_PASSWORD}', database='tomario')
with conn.cursor() as cur:
    cur.execute('SELECT id FROM bookings ORDER BY id DESC LIMIT 1')
    print('削除前の最新予約ID:', cur.fetchone())
    cur.execute('DELETE FROM bookings ORDER BY id DESC LIMIT 1')
    conn.commit()
    print('1件削除しました（障害シミュレーション）')
"
date   # 削除時刻をメモ
exit
```

> ECS Exec 経由で長い複数行コマンドを貼り付けると SSM セッションで文字化け・継続プロンプト混入が起きやすい。パスワードは短く 1 行だけ `/tmp/pw.txt` に保存し、スクリプト側で読み込む方式にすると事故りにくい。

### 手順 3（B-03）：ポイントインタイムリストア

```bash
# 削除時刻の 1 分前を JST で指定
RESTORE_TIME="2026-07-21T10:08:30+09:00"   # ← 実際の時刻に置換

aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier tomario-staging-rds \
  --target-db-instance-identifier tomario-staging-rds-restore-test \
  --restore-time "$RESTORE_TIME" \
  --db-instance-class db.t3.micro \
  --no-multi-az \
  --db-subnet-group-name tomario-staging-rds-subnet-group \
  --vpc-security-group-ids $(aws rds describe-db-instances --db-instance-identifier tomario-staging-rds \
    --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" --output text)
date   # 復元開始時刻をメモ

aws rds wait db-instance-available --db-instance-identifier tomario-staging-rds-restore-test
date   # 復元完了時刻をメモ → 開始との差分が RTO
```

### 手順 4（B-04）：復元データの検証

```bash
RESTORE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier tomario-staging-rds-restore-test \
  --query "DBInstances[0].Endpoint.Address" --output text)

# コンテナに入り直して復元先に接続（SG は元 RDS と同一のため同じタスクから到達可能）
python3 -c "
import pymysql
conn = pymysql.connect(host='$RESTORE_ENDPOINT', user='admin', password='{DB_PASSWORD}', database='tomario')
with conn.cursor() as cur:
    cur.execute('SELECT COUNT(*) FROM bookings'); print('bookings件数:', cur.fetchone())
    cur.execute('SELECT id FROM bookings ORDER BY id DESC LIMIT 1'); print('最新予約ID:', cur.fetchone())
"
```

B-02 でメモした「削除前の最新予約 ID」がここで表示されればリストア成功。

## 実施後の記録

- 結果を [../results/backup-test-result.md](../results/backup-test-result.md) の結果表に転記する
- RTO 実測値・`LatestRestorableTime` の反映ラグを記録し、非機能要件定義書の RTO / RPO 目標値へフィードバックする
- エビデンスを `../evidence/backup/` に格納する（機密情報はマスク）
