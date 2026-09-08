# 非機能試験 実施サマリ

集計日：2026-09-06
対象：[test-plan.md](test-plan.md) / [procedures/](procedures/) / [results/](results/) の全試験項目

試験項目番号は計画・手順書・結果報告書で共通。件数は手順書の試験項目行（準備・実施・後始末・任意を含む）で数えている。

---

## 1. 全体件数

| ステータス | 件数 |
|---|---|
| ✅ 完了（準備・後始末ステップ含む） | 14 |
| 🔺 一部実施 | 7 |
| ⬜ 未実施 | 23 |
| **合計** | **44** |

- 「一部実施」も未完了に含めると **未完了 30 / 完了 14**
- 純粋な確認項目のみ（準備・後始末・任意を除く）：**33 件**

### staging / production の別

| 区分 | 合計 | ✅完了 | 🔺一部 | ⬜未実施 |
|---|---|---|---|---|
| staging ＋ CI / 共通 | 34 | 14 | 7 | 13 |
| production 固有（WAF・Security Hub・AWS Config 導入後） | 10 | 0 | 0 | 10 |

---

## 2. 試験種別ごとの件数

| 種別 | 手順書 | 合計 | ✅完了 | 🔺一部 | ⬜未実施 | 実施日 |
|---|---|---|---|---|---|---|
| スケーラビリティ（P-00〜P-09） | [scalability-test-procedure.md](procedures/scalability-test-procedure.md) | 10 | 6 | 1 | 3 | 2026-07-20 |
| 可用性・信頼性（A-01〜A-05） | [availability-test-procedure.md](procedures/availability-test-procedure.md) | 5 | 3 | 0 | 2 | 2026-07-20 |
| バックアップ・復旧（B-01〜B-06） | [backup-test-procedure.md](procedures/backup-test-procedure.md) | 6 | 5 | 0 | 1 | 2026-07-21 |
| 監視（M-01〜M-08） | [monitoring-test-procedure.md](procedures/monitoring-test-procedure.md) | 8 | 0 | 3 | 5 | 2026-09-03（一部） |
| 運用オペレーション（O-01〜O-05） | [operations-test-procedure.md](procedures/operations-test-procedure.md) | 5 | 0 | 1 | 4 | — |
| セキュリティ（S-01〜S-10） | [security-test-procedure.md](procedures/security-test-procedure.md) | 10 | 0 | 2 | 8 | — |
| **合計** | | **44** | **14** | **7** | **23** | |

---

## 3. 実施済み（✅ 完了：14 件）

| 項番 | 項目 | 結果 |
|---|---|---|
| P-01 | RDS 一時スケールアップ（準備） | db.t4g.medium へ変更 |
| P-02 | k6 で負荷投入 | 50 VU / 9 分、checks 成功率 99.96% |
| P-03 | スケールアウト確認 | desiredCount 2→3→4、全件 Successful、所要 約 3 分 |
| P-04 | スケールイン確認 | 負荷停止後 約 10〜15 分で min(2) |
| P-05 | 可用性・レスポンスタイム評価 | 成功率 99.96%（≥ 99%）。p95 10.88s は参考記録 |
| P-06 | RDS を元に戻す（後始末） | db.t3.micro へ復帰済み |
| A-01 | デプロイサーキットブレーカーによる自動ロールバック | MTTR 約 1 分 50 秒、冗長数維持 |
| A-02 | タスク強制停止からの自動復旧 | 1 分未満で復帰、ALB 2 台 healthy |
| A-03 | 処理中リクエストへの影響・データ整合性 | 500 が 1 件（実害なし）、不整合レコードなし |
| B-01 | 自動バックアップの取得確認 | LatestRestorableTime が約 6 分前、正常 |
| B-02 | 障害シミュレーション（データ削除） | bookings ID:5000 を削除 |
| B-03 | ポイントインタイムリストア | RTO 約 14 分（初回は反映ラグでエラー、再実行で成功） |
| B-04 | 復元データの検証 | 削除した ID:5000 が復元先に存在 |
| B-05 | 訓練用インスタンスの削除（後始末） | 削除済み、ECS Exec も無効化 |

---

## 4. 一部実施（🔺：7 件）

| 項番 | 項目 | 実施済みの範囲 | 残り |
|---|---|---|---|
| P-07 | ダッシュボード可視化（任意） | CPU 使用率ウィジェットは表示 | RunningTaskCount 未表示（Container Insights 未有効） |
| M-03 | 閾値超過 → メール通知到達 | 手動でアラーム状態を作る発火は確認（2026-09-03） | メトリクス閾値を実際に超過させる発報試験 |
| M-05 | ECS アプリログの出力確認 | `/ecs/tomario-staging` の保持 30 日は確認 | ログ内容から直近リクエストを追う確認 |
| M-06 | メトリクスダッシュボードの視認性 | CPU 使用率は視認可 | RunningTaskCount 未表示（P-07 と同一課題） |
| O-02 | cost-stop / cost-start によるインフラ再構築 | 運用で繰り返し実施、`terraform plan` 差分ゼロを都度確認 | 「試験」としての所要時間・CloudFront 再作成有無の記録 |
| S-04 | ネットワーク境界の構成確認 | ALB 直アクセス 403（SEC-7）は staging / production で確認 | RDS 到達不可・ECS パブリック IP なし・S3 非公開のまとめ確認 |
| S-05 | 脅威検知・監査証跡の実効性 | GuardDuty・CloudTrail の有効化は確認（2026-09-03） | イベント記録・S3 保存の実体確認 |

---

## 5. 未実施（⬜：23 件）

### 5-1. staging / 共通で着手可能（13 件）

| 項番 | 項目 |
|---|---|
| P-00 | ベースライン測定（負荷なし時の応答時間・CPU / メモリ） |
| P-08 | 目標スループット達成確認（目標 rps は非機能要件へ追記後に確定） |
| A-04 | 正常なローリングデプロイ中の無停止性（5xx ≒ 0） |
| A-05 | 壊れたリビジョンの後片付け（`deregister-task-definition`） |
| B-06 | 手動スナップショットからの復元（任意） |
| M-01 | SNS サブスクリプション確認 |
| M-02 | CloudWatch アラームが OK 状態 |
| M-04 | ログ追跡性（CloudWatch Logs Insights） |
| O-01 | デプロイロールバック手順の実演 |
| O-03 | ロールバック後の復帰（後始末） |
| S-01 | 依存パッケージの脆弱性スキャン（`pip-audit`、SEC-4 の導入が前提） |
| S-02 | コンテナイメージの脆弱性スキャン |
| S-06 | IAM 最小権限の棚卸し（任意） |

### 5-2. production 固有（WAF・Security Hub・AWS Config 導入後）（10 件）

| 項番 | 項目 |
|---|---|
| P-09 | WAF 有効時のレイテンシ影響（p50 / p95 差分） |
| M-07 | WAF ログの配信確認（配信先 S3 / CloudWatch Logs / Firehose の決定・実装が前提） |
| M-08 | WAF BlockedRequests メトリクスのアラート発報 |
| O-04 | WAF Web ACL の cost-stop/start 組み込み（作成／関連付け／削除が回る） |
| O-05 | WAF 緊急デタッチ手順（誤検知時の初動ランブック） |
| S-03 | TLS 設定の確認（testssl.sh / SSL Labs） |
| S-07 | WAF マネージドルールの有効性（攻撃ペイロードのブロック） |
| S-08 | WAF 誤検知確認（COUNT モード → 除外設定 → BLOCK 切替） |
| S-09 | Security Hub の検出結果レビュー（Critical / High findings） |
| S-10 | AWS Config ルールのコンプライアンス評価（NON_COMPLIANT リソース） |

---

## 6. 次に着手すべき順（推奨）

1. 非機能要件定義書へ目標値追記（p95・RTO / RPO・目標 rps）→ P-05 / B-03 / P-08 を「目標 vs 実測」の合否表にする
2. staging / 共通の未実施 13 件のうち高 ROI から：M-01 / M-02 / M-03（発報試験）→ M-04 → O-01（ロールバック実演）→ S-01 / S-02（スキャン）
3. WAF ログ配信先を決めて実装（S-07 / S-08 / M-07 の前提）
4. WAF・Security Hub・AWS Config を production に導入 → production 固有 10 件を実施順序（`test-plan.md` 第 5 節）に沿って実施
