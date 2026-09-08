# 非機能試験（スケーラビリティ）結果報告書

## ステータス
一部実施済み（2026-07-20）。実施した検証項目（P-03 / P-04 / P-05）は全項目合格。P-00・P-08 は未実施

対応する手順書：[../procedures/scalability-test-procedure.md](../procedures/scalability-test-procedure.md)
対応する計画：[../test-plan.md](../test-plan.md) 性能試験（P）

## 対象環境・構成
- staging / ECS Fargate + Application Auto Scaling（min=2 / max=4 / target CPU 70%）
- RDS は試験中のみ db.t4g.medium へ一時スケールアップ（試験後 db.t3.micro へ復帰）
- 負荷ツール：k6（CloudFront 経由）／テストデータ users 2,000 件・bookings 5,000 件

## 結果（手順書の各項目に対応）

| 番号 | 項目 | 実測結果 | 合否 |
|---|---|---|---|
| P-00 | ベースライン測定 | 未実施（次回の負荷試験時に、負荷なしの応答時間・CPU / メモリを取得する） | ⬜ |
| P-01 | RDS 一時スケールアップ | 実施。`db.t4g.medium` へ変更完了 | — 準備 |
| P-02 | k6 で負荷投入 | 実施。50 VU / 計 9 分。k6 サマリ：`checks_succeeded 99.96%`（5,110 / 5,112）、`avg=3.89s`、`p(95)=10.88s` | — 実施 |
| P-03 | スケールアウト確認 | CPU 使用率上昇に伴い desiredCount が **2 → 3 → 4** と段階的に増加。スケーリングアクティビティは**全件 Successful**。スケールアウト所要 約 3 分 | ✅ |
| P-04 | スケールイン確認 | 負荷停止後 **約 10〜15 分**で min(2) まで自動スケールイン | ✅ |
| P-05 | 可用性・レスポンスタイム評価 | 成功率 **99.96%**（≥ 99% を満たす）。p(95)=10.88s / avg=3.89s はスケールアウトが追いつくまでの過渡的遅延で、目標値未設定のため参考記録 | ✅（成功率）／p95 は参考 |
| P-06 | RDS を元に戻す | 実施。`db.t3.micro` へ復帰済み（漏れなし） | — 後始末 |
| P-07 | ダッシュボード可視化（任意） | **未達**。`tomario-staging-autoscaling` ダッシュボードの RunningTaskCount が「データがありません」。`RunningTaskCount` は Container Insights 有効時のみ配信される `ECS/ContainerInsights` のメトリクスで、当クラスターは未有効だった。CPU 使用率ウィジェットは正常表示 | 🔺 |
| P-08 | 目標スループット達成確認 | 未実施（非機能要件へ目標 rps を追記後に実施） | ⬜ |
| P-09 | WAF 有効時のレイテンシ影響（production） | 未実施（WAF 導入後、WAF 有効 / 無効で負荷を流し p50 / p95 差分を比較） | ⬜ |

## 考察・学び
- スケールインはスケールアウトより時間がかかる。標準のスケールイン側アラームは「15 分連続で閾値未満 + クールダウン 5 分」を要するため、スケールアウトの約 3 分に対して 10〜15 分かかることを実地で確認した
- p(95)=10.88s は目標値未設定だが体感的に重い。原因はスケールアウト完了までの過渡区間であり、定常状態の遅延ではない。非機能要件にレスポンスタイム目標値（例：定常時 p95 < 500ms）を追加し、次回は P-00 のベースラインを取ってから定常状態で測り直す

## 派生した改善・課題
- **cost-start.yml の desired-count 固定バグを発見・修正**：「Scale ECS」ステップが全環境共通で `--desired-count 1` にハードコードされており、staging の設計値 2 が cost-start のたびに 1 に落とされていた。環境ごとに正しい値を使うよう修正（PR: `fix/cost-start-staging-desired-count`、`91d3dbd` でマージ済み）
- **P-07 の RunningTaskCount 未表示**：Container Insights 導入とセットで後日対応（コスト増を伴うため保留）。`monitoring-test-result.md` M-06 と同一課題
- 非機能要件定義書にレスポンスタイム・目標スループットの目標値が未設定。P-00 / P-05 / P-08 の実測をもとに追記する

## エビデンス
- `../evidence/scalability/` — k6 サマリ出力、スケーリングアクティビティ一覧（全件 Successful）、CPU 使用率ウィジェット
- ※ RunningTaskCount ダッシュボードのスクリーンショットは P-07 の課題により未取得
