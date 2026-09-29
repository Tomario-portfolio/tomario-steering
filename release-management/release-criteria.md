# リリース判定基準

最終更新：2026-09-30

`tomario-app`のstagingでの検証が完了し、productionへpromoteする（＝一般公開する）かどうかを判定するための基準。

## 1. 判定のタイミング

- staging環境でのデプロイ（`deploy.yml`の`deploy`ジョブ、`env: staging`）が成功し、動作確認が完了した後
- `promote-to-production`ジョブの実行前（GitHub Environment「prod」の承認待ちになったタイミング）

promoteは1コミットにつき1回、staging検証済みの同一イメージ（digest指定）をそのままproductionへ流す設計のため、判定はこの1箇所に集約される。

## 2. 判定基準

以下をすべて満たすことを確認してから承認する。

| 区分 | 基準 | 根拠・確認先 |
|---|---|---|
| 機能 | staging上での主要導線（会員登録→ログイン→予約→キャンセル）が手動確認で問題なし | デプロイ後の手動クリックスルー |
| 非機能・性能 | スケールアウト・レスポンスタイムが既定の目標値を満たす、または既知の乖離が説明可能 | [performance-test-result.md](../verification/non-functional-test/results/performance-test-result.md)（P-00〜P-09） |
| 非機能・可用性 | デプロイサーキットブレーカーによる自動ロールバック、無停止デプロイが機能する | [availability-test-result.md](../verification/non-functional-test/results/availability-test-result.md)（A-01〜A-04） |
| 非機能・バックアップ | 自動バックアップ・PITRが機能し、RTOが許容範囲内 | [backup-test-result.md](../verification/non-functional-test/results/backup-test-result.md)（B-01〜B-04） |
| 非機能・監視 | アラート発報〜通知到達、ログ追跡性が機能する | [monitoring-test-result.md](../verification/non-functional-test/results/monitoring-test-result.md)（M-01〜M-08） |
| セキュリティ | 依存パッケージ・コンテナイメージにCritical/High脆弱性が残っていない（許容判断済みのものを除く） | [security-test-result.md](../verification/non-functional-test/results/security-test-result.md)（S-01・S-02） |
| セキュリティ | ネットワーク境界（ALB直アクセス拒否・RDS非公開等）が設計通り | 同上（S-04・S-05） |
| 運用 | ロールバック手順が実演済みで、実際に機能することを確認済み | [operations-test-result.md](../verification/non-functional-test/results/operations-test-result.md)（O-01・O-02） |

Tomarioでは上記すべてが2026-09-29時点で決着済み（全43項目：合格36件・意図的スキップ7件、詳細は[test-summary.md](../verification/non-functional-test/test-summary.md)）。**この状態が「公開してよい」の実体的な条件**であり、以降の新規変更に対しても同じ基準を再適用する。

## 3. 未解決の指摘があった場合の扱い

- **Critical/High相当の指摘が残っている**：判定不可（承認しない）。修正・再検証してから再判定
- **Medium以下、または「実害なし」と判断済みの指摘が残っている**（例：S-02のDebianパッチ待ちHIGH2件）：判定理由を明記した上で承認可。判断の根拠は結果報告書側に残す
- **非機能試験の目標値を厳密には満たさないが、原因を特定済みで許容できる**（例：P-08のp95/rps未達を、原因（Auto Scalingの反応時間）を理解した上で合格扱いにしたケース）：同様に判定理由を明記した上で承認可

## 4. 関係者

| 役割 | 担当 | やること |
|---|---|---|
| リリース判定者 | プロジェクトオーナー（本ポートフォリオでは自分自身が兼務） | 上記基準の充足確認、GitHub Environment「prod」でのpromote承認 |
| 実施者 | デプロイ担当（同上） | promote実行、[cutover-and-rollback.md](cutover-and-rollback.md)に沿った切替作業 |
| 監視担当 | 運用担当（同上） | [post-release-checklist.md](post-release-checklist.md)・[hypercare-and-handover.md](hypercare-and-handover.md)に沿った監視 |

実務では判定者・実施者・監視担当が別人であることが前提（承認者と実施者の分離）。本ポートフォリオでは1人で兼務するが、GitHub Environmentの承認ゲートという形で「判定と実行の間に必ず承認ステップを挟む」構造自体は再現している。
