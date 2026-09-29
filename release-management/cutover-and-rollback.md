# カットオーバー手順・切り戻し手順

最終更新：2026-09-30

[release-criteria.md](release-criteria.md)の基準を満たしたあと、実際にproductionへ切り替える手順と、問題発生時に切り戻す手順。

前提：production環境は`cost-start`済み（停止中の場合は先に起動する）。計画メンテナンスウィンドウを設け、事前に切替予定時刻を関係者へ共有しておく。

## 1. カットオーバー手順

1. **事前確認**：[release-criteria.md](release-criteria.md)の判定基準を満たしていることを確認する
2. **promote実行**：`tomario-app`の`deploy.yml`ワークフローで、`promote-to-production`ジョブがGitHub Environment「prod」の承認待ちになっていることを確認し、承認する
   - staging検証済みイメージをdigest指定でpull → 再タグ → `tomario-production-app`（productionのECRリポジトリ）へpush（再ビルドはしない。stagingとproductionで完全に同一のイメージを使う）
   - production ECSタスク定義を、そのdigestのイメージで更新するデプロイジョブが自動実行される
3. **フロントエンド反映**：`deploy-frontend-production`ジョブでS3への同期・CloudFrontキャッシュのinvalidateが実行される（`promote-to-production`と同じワークフロー内、自動）
4. **デプロイ完了確認**：ECSサービスのデプロイが`COMPLETED`になることを確認する（デプロイサーキットブレーカーが有効なため、異常があれば自動的にロールバックされる。[A-01](../verification/non-functional-test/results/availability-test-result.md)参照）
5. **稼働確認**：[post-release-checklist.md](post-release-checklist.md)を実施する

## 2. 切り戻し手順

### 2.1 デプロイ中の異常（自動）

ECSのデプロイサーキットブレーカーが、新タスクのヘルスチェック失敗を検知すると自動的に直前のタスク定義へロールバックする。手動対応は不要（[A-01](../verification/non-functional-test/results/availability-test-result.md)で実測、MTTR約1分50秒）。

### 2.2 デプロイ完了後に問題が判明した場合（手動）

[operations-test-result.md](../verification/non-functional-test/results/operations-test-result.md)のO-01・O-02で実演済みの手順：

1. ECSサービスの現在のタスク定義リビジョン番号を確認する
2. 1つ前の正常だったタスク定義リビジョンを指定し、`update-service`でデプロイし直す（`aws ecs update-service --cluster <cluster> --service <service> --task-definition <family>:<前のリビジョン>`）
3. デプロイが`COMPLETED`になるまで待つ（O-01実測：所要約3分10秒）
4. 切り戻し後、正常なリビジョンで動作していることを確認する（O-02実測：復帰まで約3分8秒）

イメージはdigest指定でstagingと完全同一のものを使っているため、「1つ前のdigestへ戻す」＝「1つ前のタスク定義リビジョンへ戻す」で完結する（再ビルド・再pushは不要）。

### 2.3 データベースに影響する変更だった場合

マイグレーションを伴うリリースでロールバックが必要な場合、アプリのタスク定義を戻すだけでは不十分になりうる。その場合は[../verification/non-functional-test/results/backup-test-result.md](../verification/non-functional-test/results/backup-test-result.md)のB-03（ポイントインタイムリストア、RTO約14分）の手順を用いる。ただし戻す先の時点以降のデータは失われるため、実施判断はリリース判定者が行う。

### 2.4 公開停止が必要な場合（緊急時）

WAFのブロックルールに起因する誤検知等で緊急にアクセス制御を変更する必要がある場合は、[O-03（WAF緊急デタッチ手順）](../verification/non-functional-test/results/operations-test-result.md)を参照（デタッチ〜復旧約1分20秒で実演済み）。
