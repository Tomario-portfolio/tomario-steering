# evidence/

各試験のエビデンス（スクリーンショット・コマンド出力・ログ抜粋）の置き場。
結果報告書の項番に対応するサブフォルダに格納する。

```
evidence/
  scalability/   … k6 サマリ出力、スケーリングアクティビティ一覧、CPU 使用率ウィジェット
  availability/  … デプロイイベント（FAILED→ロールバック）、describe-services の推移、該当エラーログ、rolling_deploy_health.log
  backup/        … LatestRestorableTime 確認出力、リストア実行ログ、復元先での件数確認
  monitoring/    … アラーム一覧、ALARM 遷移履歴、メール受信画面、Logs Insights 結果（未取得）
  security/      … 依存/イメージスキャンレポート、testssl.sh 出力、境界確認コマンド結果（未取得）
  operations/    … describe-services の状態遷移、terraform plan 差分ゼロ出力、疎通確認結果（未取得）
```

サブフォルダ名は `procedures/` / `results/` のファイル名と対応（`scalability-test-procedure.md` ⇔ `scalability-test-result.md` ⇔ `evidence/scalability/`）。

## 注意
- 機密情報（DB エンドポイント、パスワード、アカウント ID 等）はマスクしてから格納する
- 現時点では過去試験（2026-07）のエビデンスは未収集。再取得できるものは cost-start 時に取得し、
  再現できないものは当時のコマンド出力メモ（`tomario-workspace/reference/test/staging/`）から転記する
