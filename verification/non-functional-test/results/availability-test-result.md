# 非機能試験（可用性）結果報告書

## ステータス
実施済み。A-01〜A-04 全項目合格（A-04は2026-09-26追加）。

対応する手順書：[../procedures/availability-test-procedure.md](../procedures/availability-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 可用性・信頼性試験（A）

## 対象環境・構成
- staging / ECS Fargate、desired_count=2、デプロイサーキットブレーカー有効
- ALB ターゲットグループ `deregistration_delay=30`

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| A-01 | デプロイサーキットブレーカーによる自動ロールバック | 存在しないコマンドを仕込んだ revision 2 をデプロイ → `FAILED`（reason: tasks failed to start）→ 自動的に revision 1 へロールバック。**MTTR 約 1 分 50 秒**（FAILED 検知 16:13:37 → ロールバック完了 16:15:25）。ロールバック中も `desired:2 / running:2` を維持 | ✅ |
| A-02 | タスク強制停止からの自動復旧 | 稼働中タスクを 1 つ `stop-task` → **1 分未満**で代替タスクが自動起動し `desired:2 / running:2` に復帰。ALB ターゲットグループも 2 台とも healthy | ✅ |
| A-03 | 処理中リクエストへの影響・データ整合性 | 予約作成 API へ 100 リクエスト送信中にタスクを強制停止。`500` が 1 件（タスク停止で DB 接続断。`pymysql.err.OperationalError (2013, 'Lost connection to MySQL server during query')`。予約 INSERT 本体ではなく手前のログインユーザー確認 SELECT で発生＝**書き込み前に失敗しており実害なし**）、`400` が 99 件（同一 room_id / 日程の重複予約エラーで、タスク停止とは無関係）、`201` は 0 件。DB 確認：本試験で新規作成された予約は 0 件、中途半端なレコードなし | ✅ |
| A-04 | 正常なローリングデプロイ中の無停止性 | **合格**（2026-09-26）。`/health`へ0.3秒間隔で400回curlしながら`force-new-deployment`（タスク定義は変更なし）を実行。**400件全て200**、5xxゼロ | ✅ |

## 考察・学び
- サーキットブレーカーはロールバック中も冗長数を維持するため、不正デプロイによるサービス断は発生しなかった
- タスク停止による `500` は「書き込み処理のどの段階で失敗したか」で実害の有無が変わる。今回はトランザクション開始前の SELECT だったため不整合ゼロ。書き込み中に落ちるケースの整合性はアプリ側のトランザクション設計に依存する
- A-01（異常系）に加えてA-04（正常デプロイ時の無停止性）も確認でき、異常系・正常系の両方でデプロイ中の可用性が保たれることを実測できた

## 派生した改善・課題
- 特になし

## エビデンス
- `../evidence/availability/` — デプロイイベント（FAILED → ロールバック）、`describe-services` の推移、CloudWatch Logs の該当エラー行、（A-04 実施時）`rolling_deploy_health.log`
