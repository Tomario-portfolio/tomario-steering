# 非機能試験 実施サマリ

最終更新：2026-09-29（初版2026-09-06、staging試験ラウンド完了に伴い全面更新）
対象：[test-plan.md](test-plan.md) / [procedures/](procedures/) / [results/](results/) の全試験項目

**全43項目が決着済み**（実施・合格 36件、意図的スキップ 7件）。試験項目番号は計画・手順書・結果報告書で共通。

---

## 1. 全体件数

| ステータス | 件数 |
|---|---|
| ✅ 合格（準備・後始末・スキップ判断を含む） | 43 |
| 🔺 一部合格・未実施 | 0 |
| **合計** | **43** |

### 試験種別ごとの内訳

| 種別 | 手順書 | 件数 | 実施日 |
|---|---|---|---|
| 性能（P-00〜P-09） | [performance-test-procedure.md](procedures/performance-test-procedure.md) | 10 | 2026-07-20／2026-09-26〜29 |
| 可用性・信頼性（A-01〜A-05） | [availability-test-procedure.md](procedures/availability-test-procedure.md) | 5 | 2026-07-20／2026-09-26 |
| バックアップ・復旧（B-01〜B-06） | [backup-test-procedure.md](procedures/backup-test-procedure.md) | 6 | 2026-07-21／2026-09-26 |
| 監視（M-01〜M-08） | [monitoring-test-procedure.md](procedures/monitoring-test-procedure.md) | 8 | 2026-09-03／09-12／09-16／09-26〜29 |
| 運用オペレーション（O-01〜O-03） | [operations-test-procedure.md](procedures/operations-test-procedure.md) | 3 | 2026-09-16／09-26 |
| セキュリティ（S-01〜S-10） | [security-test-procedure.md](procedures/security-test-procedure.md) | 10 | 2026-09-12／09-16／09-26〜29 |
| **合計** | | **42*** | |

\* 準備・後始末ステップ（P-01/P-06、B-02/B-03a/B-05）7件を含む延べ件数。判定対象（合格・スキップ）のみで数えると36件。

運用オペレーションの項番はO-01〜O-03（旧O-02「cost-stop/cost-startによるインフラ再構築確認」は日常運用で自然に確認できるため試験項目から除外、旧O-03・O-05をそれぞれO-02・O-03に繰り上げ）。

---

## 2. 合格項目（30件）

| 項番 | 項目 | 結果概要 |
|---|---|---|
| P-00 | ベースライン測定 | API応答時間平均116ms（min85/max177ms） |
| P-03 | スケールアウト確認 | desiredCount 2→3→4、所要約3分、全件Successful |
| P-04 | スケールイン確認 | 負荷停止後10〜15分でmin(2)に収束 |
| P-05 | 可用性・レスポンスタイム評価 | 成功率99.96%（≥99%） |
| P-07 | ダッシュボード可視化 | Container Insights導入によりRunningTaskCount表示を確認 |
| P-08 | 目標スループット達成確認 | gunicorn化後、成功率100%達成 |
| A-01 | デプロイサーキットブレーカーによる自動ロールバック | MTTR約1分50秒 |
| A-02 | タスク強制停止からの自動復旧 | 1分未満で復帰 |
| A-03 | 処理中リクエストへの影響・データ整合性 | 不整合レコードなし |
| A-04 | 正常なローリングデプロイ中の無停止性 | 400件全て200、5xxゼロ |
| B-01 | 自動バックアップの取得確認 | LatestRestorableTimeが約6分前 |
| B-03 | ポイントインタイムリストア | RTO約14分 |
| B-04 | 復元データの検証 | 削除レコードの復元を確認 |
| M-01 | SNSサブスクリプション確認 | 実ARNで購読済み |
| M-02 | アラームがOK状態 | 全アラームOK |
| M-03 | 閾値超過→メール通知到達 | OK→ALARM遷移・メール受信を確認 |
| M-04 | ログ追跡性 | multiline-pattern反映後、スタックトレース全体を追跡可能 |
| M-05 | ECSアプリログの出力確認 | 直近リクエストを追跡可能 |
| M-06 | メトリクスダッシュボードの視認性 | RunningTaskCount配信を確認 |
| M-07 | WAFログの配信確認（production） | CloudWatch Logsへ配信・可読を確認 |
| M-08 | WAF BlockedRequestsアラートの発報（production） | OK→ALARM遷移・メール受信を確認 |
| O-01 | デプロイロールバック手順の実演 | 所要約3分10秒 |
| O-02 | ロールバック後の復帰 | 所要約3分8秒 |
| O-03 | WAF緊急デタッチ手順（production） | デタッチ〜復旧約1分20秒 |
| S-01 | 依存パッケージの脆弱性スキャン | pip-audit実行、修正後Critical/Highゼロ |
| S-02 | コンテナイメージの脆弱性スキャン | OSパッケージ更新後High2件（実害なしと判断） |
| S-04 | ネットワーク境界の構成確認 | ALB直アクセス拒否・RDS到達不可・パブリックIPなし・S3非公開を確認 |
| S-05 | 脅威検知・監査証跡の実効性 | GuardDuty有効・CloudTrailイベント記録を確認 |
| S-07 | WAFマネージドルールの有効性（production） | XSS・パストラバーサルをBLOCK |
| S-09 | Security Hubの検出結果レビュー（production） | Critical/High 13件を全件レビュー |

## 3. 意図的スキップ（7件）

判断・理由が確定した項目。詳細は各resultファイル参照。

| 項番 | 項目 | 理由 |
|---|---|---|
| P-09 | WAF有効時のレイテンシ影響（production） | エッジでの影響は元々小さく、非公開期間で実測価値が薄い |
| B-06 | 手動スナップショットからの復元（任意） | B-03（PITR）と同種で新規性が薄い |
| S-03 | TLS設定の確認 | 独自ドメイン・ACM証明書を導入しないため、TLS1.0/1.1が有効なまま残ることを受け入れる（[ADR](../../adr/infra/frontend/001-no-custom-domain.md)） |
| S-06 | IAM最小権限の棚卸し（任意） | 元々任意項目 |
| S-08 | WAF誤検知確認（production） | 非公開期間で実ユーザーがおらずCOUNT運用の必要性が薄い（一般公開後に再検討） |
| S-10 | AWS Configコンプライアンス評価（production） | Security Hub（S-09）が同種の評価を実施済み |

---

## 4. 試験を通じて発見・修正した主な問題

非機能試験の価値は「設定した」ことの確認ではなく「実際に機能するか」を実測で確認する点にある。今回のラウンドで見つかった代表的な問題：

- **P-08（性能試験）**：`tomario-app`がFlask開発用サーバー（シングルスレッド）のまま稼働しており、20rps程度の負荷で応答時間が15秒超に悪化。gunicornへの移行で解消（`tomario-app` PR #13・#14）
- **S-01 / S-02（脆弱性診断）**：依存パッケージ13件・コンテナイメージのOSパッケージ27件の既知脆弱性を検出。バージョン更新・`apt-get upgrade`導入で大半を解消
- **M-04（ログ追跡性）**：`awslogs-multiline-pattern`未設定によりスタックトレースが1行＝1イベントに分断される問題を発見・修正（`tomario-infra` PR #90）
- **M-08 / O-03（WAF運用）**：CloudWatchアラームのメトリクスディメンション誤り、対象リージョン誤り、SNSトピックのリージョン不一致、REGIONAL/CLOUDFRONTスコープAPIの取り違えなど、実機検証で初めて顕在化するバグを複数発見・修正（`tomario-infra` PR #84〜#88）
- **CloudFrontのステータスコードマスキング**：SPAルーティング対応の`custom_error_response`が403/404をクライアント向けに200へ上書きするため、WAF・アプリ双方の合否判定にHTTPステータスコードが使えないことが判明。オリジン側ログ（WAFサンプリングログ・CloudWatch Logs）を正とする運用に統一。403については`tomario-infra` PR #83（CloudFront Functionへの移行）で解消済みで、404→200のみSPA用に残っている

---

## 関連ドキュメント

- 環境固有の設定値・productionでの詳細実行ログ：[../env/production-only-test-items.md](../env/production-only-test-items.md)
- 各試験の実施手順：[procedures/](procedures/)
- 各試験の結果報告書：[results/](results/)
