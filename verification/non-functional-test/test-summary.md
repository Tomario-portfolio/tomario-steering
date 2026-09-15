# 非機能試験 実施サマリ

集計日：2026-09-16（初版2026-09-06、production固有9件の消化に伴い更新）
対象：[test-plan.md](test-plan.md) / [procedures/](procedures/) / [results/](results/) の全試験項目

試験項目番号は計画・手順書・結果報告書で共通。件数は手順書の試験項目行（準備・実施・後始末・任意を含む）で数えている。
「スキップ」（意図的に実施しないと判断した項目）は、判断・理由が確定した時点で「完了」に含めてカウントする。

---

## 1. 全体件数

| ステータス | 件数 |
|---|---|
| ✅ 完了（準備・後始末ステップ、スキップ判断済みを含む） | 22 |
| 🔺 一部実施 | 7 |
| ⬜ 未実施 | 13 |
| **合計** | **42** |

- 「一部実施」も未完了に含めると **未完了 20 / 完了 22**
- 純粋な確認項目のみ（準備・後始末・任意を除く）：**31 件**
- S-03（TLS設定の確認、production固有）はユーザー判断で項目自体を削除したため、母数が43→42に変わっている

### staging / production の別

| 区分 | 合計 | ✅完了 | 🔺一部 | ⬜未実施 |
|---|---|---|---|---|
| staging ＋ CI / 共通 | 34 | 14 | 7 | 13 |
| production 固有（WAF・Security Hub・AWS Config 導入後） | 8 | 8 | 0 | 0 |

**production固有はこれで全項目決着済み**（詳細は[../env/production-only-test-items.md](../env/production-only-test-items.md)）。

---

## 2. 試験種別ごとの件数

| 種別 | 手順書 | 合計 | ✅完了 | 🔺一部 | ⬜未実施 | 実施日 |
|---|---|---|---|---|---|---|
| スケーラビリティ（P-00〜P-09） | [scalability-test-procedure.md](procedures/scalability-test-procedure.md) | 10 | 7 | 1 | 2 | 2026-07-20（P-09スキップは2026-09-16） |
| 可用性・信頼性（A-01〜A-05） | [availability-test-procedure.md](procedures/availability-test-procedure.md) | 5 | 3 | 0 | 2 | 2026-07-20 |
| バックアップ・復旧（B-01〜B-06） | [backup-test-procedure.md](procedures/backup-test-procedure.md) | 6 | 5 | 0 | 1 | 2026-07-21 |
| 監視（M-01〜M-08） | [monitoring-test-procedure.md](procedures/monitoring-test-procedure.md) | 8 | 2 | 3 | 3 | 2026-09-03（一部）／2026-09-12（M-07）／2026-09-16（M-08） |
| 運用オペレーション（O-01・O-02・O-03・O-05） | [operations-test-procedure.md](procedures/operations-test-procedure.md) | 4 | 1 | 1 | 2 | 2026-09-16（O-05） |
| セキュリティ（S-01〜S-10、S-03除く） | [security-test-procedure.md](procedures/security-test-procedure.md) | 9 | 4 | 2 | 3 | 2026-09-12（S-07一部・S-09完了）／2026-09-16（S-07合格・S-08/S-10スキップ確定） |
| **合計** | | **42** | **22** | **7** | **13** | |

---

## 3. 実施済み（✅ 完了：22 件）

| 項番 | 項目 | 結果 |
|---|---|---|
| M-07 | WAF ログの配信確認（production） | CloudWatch Logsへ配信済み。`action`・`httpRequest`等が読めることを確認（2026-09-12） |
| M-08 | WAF BlockedRequests メトリクスのアラート発報（production） | 攻撃ペイロード30リクエストで`OK→ALARM`遷移・SNSメール通知の受信まで実機確認。実装過程でRegionディメンション誤り・アラームのリージョン誤り・SNSトピックのリージョン不一致の3件のバグを発見・修正（tomario-infra PR #84〜#88、2026-09-16） |
| O-05 | WAF 緊急デタッチ手順（production） | CloudFrontの`update-distribution`でデタッチ→復旧確認 約1分20秒、再アタッチ完了まで通しで約3分24秒と実測。手順書記載の`wafv2 associate/disassociate-web-acl`はREGIONALスコープ専用APIでCloudFrontには使えないバグを発見・訂正（2026-09-16） |
| S-07 | WAF マネージドルールの有効性（production） | XSS・パストラバーサルを`CommonRuleSet`が`BLOCK`することを確認。SQLi対象外は意図した設計判断（ORM経由でパラメータ化済み）と整理し合格に格上げ。PR #83後はcurl応答が直接403になることも追加確認（2026-09-12／2026-09-16） |
| S-08 | WAF 誤検知（false positive）確認（production・スキップ） | `modules/waf`にCOUNT/BLOCK切替えの仕組みが未実装。production非公開期間で実ユーザーがいないためCOUNT運用の必要性が薄く、BLOCKモード直有効化＋手動クリックスルー確認で代替（`task-and-flow/remaining-task.md` #17、2026-09-12決定） |
| S-09 | Security Hub の検出結果レビュー（production） | Critical/High 13件を全件レビューし対応方針を記録（`task-and-flow/report/2026-09-12_security-hub-findings-review.md`、2026-09-12） |
| S-10 | AWS Config ルールのコンプライアンス評価（production・スキップ） | Config Rule自体は未実装だが、Security Hub（S-09）のCIS/FSBP標準が内部的に同種の評価を実施済みと判断し、重複するConfig Ruleの追加は見送り（2026-09-16決定） |
| P-09 | WAF 有効時のレイテンシ影響（production・スキップ） | CloudFrontエッジでの影響は元々小さいことが知られている上、production非公開期間で実測の価値が薄いため見送り（2026-09-16決定） |
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

## 5. 未実施（⬜：13 件、すべて staging / 共通）

production固有の項目は2026-09-16時点で全て決着済み（[3. 実施済み](#3-実施済み✅-完了22-件)参照）。残る未実施はstaging/共通のみ。

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

---

## 6. 次に着手すべき順（推奨）

1. 非機能要件定義書へ目標値追記（p95・RTO / RPO・目標 rps）→ P-05 / B-03 / P-08 を「目標 vs 実測」の合否表にする
2. staging / 共通の未実施 13 件のうち高 ROI から：M-01 / M-02 / M-03（発報試験）→ M-04 → O-01（ロールバック実演）→ S-01 / S-02（スキャン）
3. ~~WAF ログ配信先を決めて実装（S-07 / S-08 / M-07 の前提）~~ 完了
4. ~~WAF・Security Hub・AWS Config を production に導入 → production 固有 9 件を実施順序（`test-plan.md` 第 5 節）に沿って実施~~ 完了（2026-09-16、production固有は全項目決着）
