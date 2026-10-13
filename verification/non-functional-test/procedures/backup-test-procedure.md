# 非機能試験（バックアップ）実施手順書

対応する計画：[../test-plan.md](../test-plan.md) の復旧試験（B-01・B-02）
対応する結果報告書：[../results/backup-test-result.md](../results/backup-test-result.md)

## 試験実施目的

`backup_retention_period = 7` を設定していることと、実際にリストアできることは別問題である。
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
| 注意 | ポイントインタイムリストアは新規インスタンスとして復元される。訓練後は必ず削除する（B-02 の後始末） |

## 試験項目一覧

| 番号 | 項目 | 実施環境 | 試験手順 | 想定結果 | 備考 |
|---|---|---|---|---|---|
| B-01 | 自動バックアップが取得されていること | staging | 保持期間と最新の復元可能時刻を確認する（詳細手順1を参照） | `BackupRetention` が 7、`LatestRestorableTime` が現在時刻の数分〜十数分前 | 古い日時で止まっていたら取得できていないサイン |
| B-02 | PITRで削除データを復旧できること | staging | テストデータを削除し、削除前へ復元し、データとRTOを確認する。訓練用DBの削除もこの項目の後始末に含める（詳細手順2を参照） | 削除したレコードが復元先に存在する。RTO < 30 分。訓練用インスタンスが削除される | 初回は `LatestRestorableTime` 未到達で `InvalidParameterValue` になることがある（数分待って再実行） |

## 詳細手順

### 手順 1（B-01）：自動バックアップの取得確認

1. RDSインスタンスのバックアップ設定と、復元できる最新の時刻を取得する
2. バックアップの保持期間が7日であることを確認する
3. 復元できる最新の時刻が、現在時刻の数分〜十数分前であることを確認する

```bash
aws rds describe-db-instances --db-instance-identifier tomario-staging-rds \
  --query "DBInstances[0].{BackupRetention:BackupRetentionPeriod,LatestRestorableTime:LatestRestorableTime,BackupWindow:PreferredBackupWindow}"
```

### 手順 2（B-02）：PITRによる削除データの復旧

#### 2-1. テストデータの削除

ECS Exec でコンテナに入り、`bookings` の最新レコードを1件削除する。削除前の最新IDと削除時刻を記録する。削除時刻は、削除後の時点を復元ポイントにしないための基準になる。

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

#### 2-2. ポイントインタイムリストア

削除時刻の1分前を指定し、新規インスタンスとして復元する。復元の開始時刻と完了時刻を記録し、その差をRTOとする。

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

#### 2-3. 復元データの検証

復元先に接続し、件数と最新IDを確認する。

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

2-1 でメモした「削除前の最新予約 ID」がここで表示されればリストア成功。

#### 2-4. 訓練用インスタンスの削除（後始末）

1. リストアで作成した訓練用のインスタンスを、最終スナップショットを取らずに削除する
2. 数分後に、訓練用のインスタンスが存在しないことを確認する

放置するとストレージ課金が続くため、必ず実施する。

```bash
aws rds delete-db-instance --db-instance-identifier tomario-staging-rds-restore-test --skip-final-snapshot

# 数分後に実行し、DBInstanceNotFound が返れば削除完了
aws rds describe-db-instances --db-instance-identifier tomario-staging-rds-restore-test
```

## 実施後の記録

- 結果を [../results/backup-test-result.md](../results/backup-test-result.md) の結果表に転記する
- RTO 実測値・`LatestRestorableTime` の反映ラグを記録し、非機能要件定義書の RTO / RPO 目標値へフィードバックする
- エビデンスを `../evidence/backup/` に格納する（機密情報はマスク）
