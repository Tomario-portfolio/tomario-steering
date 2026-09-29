# 公開後の稼働確認チェックリスト

最終更新：2026-09-30

[cutover-and-rollback.md](cutover-and-rollback.md)のカットオーバー完了直後、公開して問題ないことを確認するためのチェックリスト。上から順に実施し、途中で異常が見つかった場合は[cutover-and-rollback.md](cutover-and-rollback.md)の切り戻し手順に従う。

## 1. ヘルスチェック

- [ ] ECSサービスのRunning task数が期待値（desiredCount）と一致している
- [ ] ALBのターゲットグループでタスクが`healthy`になっている
- [ ] CloudFront経由でフロントエンド（S3静的サイト）にアクセスできる
- [ ] CloudFront経由で`/api/*`がALB→ECSに到達し、レスポンスが返る

## 2. 主要導線の動作確認

- [ ] 会員登録ができる
- [ ] ログインができる
- [ ] ホテル予約ができる
- [ ] 予約のキャンセルができる

（[test-plan.md](../verification/non-functional-test/test-plan.md)のP-00ベースライン測定・A-03データ整合性確認と同じ導線を使う）

## 3. 監視ダッシュボード確認

- [ ] CloudWatchダッシュボードでRunningTaskCount・CPU/メモリ使用率が正常範囲内
- [ ] RDSのCPU使用率・接続数が正常範囲内
- [ ] 直近のCloudWatchアラームがすべてOK状態（[M-02](../verification/non-functional-test/results/monitoring-test-result.md)参照）

## 4. エラーログ確認

- [ ] ECSアプリログ（CloudWatch Logs）に5xxエラー・例外スタックトレースが出ていないか確認
- [ ] ALBアクセスログで5xx応答が急増していないか確認
- [ ] WAFログ（production）で想定外のBLOCKが発生していないか確認（[M-07](../verification/non-functional-test/results/monitoring-test-result.md)参照）

## 5. 完了判断

- [ ] 上記すべてに問題がない場合、カットオーバー完了とし[hypercare-and-handover.md](hypercare-and-handover.md)の監視体制へ移行する
- [ ] いずれかに問題がある場合、影響範囲を判断のうえ[cutover-and-rollback.md](cutover-and-rollback.md)の切り戻し手順を実施する
