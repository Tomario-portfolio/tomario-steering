# 003. RDS Multi-AZの見送り

## ステータス
承認済み（将来見直しの余地あり）

## コンテキスト
RDS（MySQL）はAZ障害時の単一障害点になりうる。Multi-AZ構成にすれば自動フェイルオーバーで可用性を上げられるが、インスタンス料金がほぼ倍になる（db.t3.microでも~$19/月→~$38/月程度）。production構築時に一度「productionだけMulti-AZを有効化する」方針で検討が進んだ時期があったが、最終的に全環境でSingle-AZのまま運用する決定に至った。

## 決定
dev・staging・production全環境でRDSをSingle-AZ構成のまま運用する。`multi_az`変数は用意するが、デフォルト値`false`から変更しない。

## 選定理由
- **Multi-AZ有効化（production限定）**：自動フェイルオーバーにより可用性・信頼性の実績として語れる材料になるが、非商用のポートフォリオでは実際にAZ障害が発生する確率に対してコスト増（月$15〜20程度）が見合わないと判断
- **Single-AZ継続**：AZ障害時はポイントインタイムリストア（PITR）で別AZに復元する運用で代替する。RTO30分の目標値内に収まることをstagingで実測済みのため、Multi-AZ無しでも許容範囲の復旧手段を持てていると判断

## 利点
- インスタンス料金を抑えられる（全環境Single-AZのまま）
- `multi_az`を変数化しているため、将来必要になった場合は値を`true`にしてapplyするだけで有効化できる

## 欠点
- AZ障害発生時はPITRによる復元（約14分の実測RTO＋障害検知・切替作業の時間）が必要で、Multi-AZの自動フェイルオーバー（数十秒〜数分）より復旧に時間がかかる
- AZ障害発生からリストア完了までの間、障害発生時点以降のデータ（RPO約15分未満のデータ）は失われる可能性がある

## 関連情報
- 設計の詳細：[availability-high-level-spec.md](../../../../tomario-docs/basic-design/availability-high-level-spec.md)・[database-high-level-spec.md](../../../../tomario-docs/basic-design/database-high-level-spec.md)
- PITR実測：[backup-high-level-spec.md](../../../../tomario-docs/basic-design/backup-high-level-spec.md)（staging訓練、2026-07-21実施、RTO約14分）
