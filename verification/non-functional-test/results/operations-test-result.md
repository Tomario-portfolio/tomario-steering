# 非機能試験（運用：ロールバック）結果報告書

## ステータス
全項目実施済み。O-01（staging、2026-09-26）・O-02（staging、2026-09-26）・O-03（production、2026-09-16）すべて合格。

**cost-stop / cost-start によるインフラ再構築確認（旧O-02）は2026-09-27に試験項目から除外**（`test-plan.md`参照）。cost-stop/startを日常運用として繰り返す中で自然に確認できる内容であり、旧O-04（WAF Web ACLのcost-stop/start組み込み）と同じ理由。所要時間・CloudFront再作成有無は下記「補足」に運用メモとして残す。

対応する手順書：[../procedures/operations-test-procedure.md](../procedures/operations-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 運用試験（オペレーション）（O）

## 対象環境・構成
- staging（`tomario-staging-*`）。O-01 は production の promote フローにも読み替え可能

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| O-01 | デプロイロールバック手順の実演 | **合格**（2026-09-26）。`tomario-staging-task:9→6`へ`update-service --force-new-deployment`→`wait services-stable`。所要**約3分10秒**（14:20:36開始→14:23:46完了）、`running`が0になる瞬間なし、`rolloutState:COMPLETED`、`/health`・`/api/rooms`とも200 | ✅ |
| O-02 | ロールバック後の復帰（後始末） | **合格**（2026-09-26）。O-01実施後、`tomario-staging-task:6→9`へ復帰。所要**約3分8秒**（14:24:22開始→14:27:30完了）、`rolloutState:COMPLETED`、`/health`は200 | ✅ 後始末 |
| O-03 | WAF 緊急デタッチ手順（production） | **合格**。CloudFrontの`update-distribution`でデタッチ→復旧確認 約1分20秒、再アタッチ完了まで通しで約3分24秒と実測。手順書記載の`wafv2 associate/disassociate-web-acl`はREGIONALスコープ専用APIでCloudFrontには使えないバグを発見・訂正（2026-09-16、`test-summary.md`と同期） | ✅ |

## 補足
- （運用メモ、旧O-02）cost-stop の destroy 連鎖で CloudFront も再作成対象になり、cost-start のたびに CloudFront の新規作成に 20〜30 分かかることがある（`tomario-workspace/reference/test/staging/output.md` に記録あり）
- O-01：production では staging で検証済みイメージの digest promote を実装済みのため、その逆順（1 つ前の digest への re-tag → タスク定義更新）が切り戻し手順になる

## 派生した改善・課題
- cost-stop の CloudFront 道連れ destroy は既知の非効率。cost-start の所要時間を押し上げている（別途対応検討）
- O-01実施の過程で、cost-start直後は`tomario-staging-task`の1つ前のrevisionが`bootstrap`プレースホルダー画像（ECR上に実体が無い）になっており、ロールバック先として使えないことを再確認（`remaining-task.md`記載の既知仕様）。切り戻し訓練を行う際は「1つ前」ではなく、実イメージが入っている直近のrevisionを`describe-task-definition`で確認してから選ぶ必要がある

## エビデンス
- `../evidence/operations/` — 未取得（`describe-services` の状態遷移、疎通確認結果を実施時に格納）
