# 001. cost-stop/startによる使う時だけ起動する運用

## ステータス
承認済み

## コンテキスト
本システムは非公開のポートフォリオ用途であり、実ユーザーからの常時アクセスが無い。一方でALB・ECSタスク・VPCエンドポイント（Interface型）・RDSは起動しているだけで時間課金が発生する。dev/staging/production共通で、インフラを常時起動したままにするか、必要な時だけ起動する運用にするかを決める必要があった。

## 決定
GitHub Actionsの`cost-stop.yml`/`cost-start.yml`ワークフローで、ALB・VPCエンドポイント（Interface型）の削除/作成とECSタスク数・RDSの停止/起動をまとめて行う「使う時だけ起動する」運用とする。production環境も、一般公開して常時アクセスが発生するようになるまでは、dev/staging同様この運用に含める。

## 選定理由
- **常時起動**：いつでも即座にアクセスできるが、実ユーザーがいない期間も課金が発生し続ける（nonprod/production合計で月$30〜40程度）
- **使う時だけ起動**：作業・デモ・面接の直前に起動する一手間は増えるが、未使用時のコストをほぼゼロ（RDSストレージ分のみ）に抑えられる。起動も数分で完了するため実用上の支障は小さい

## 利点
- 未使用時のコストをほぼゼロに抑えられる
- 「使う時だけ作る」という考え方を、ALB・VPCエンドポイントだけでなくWAF・Security Hub・AWS Config等の時間按分課金リソースにも応用できる（[ADR: セキュリティスタックの「使う時だけ有効化」運用](../security/001-security-stack-cost-management.md)参照）

## 欠点
- RDSは停止しても7日でAWSにより自動的に再起動される制約があり、放置すると意図せず課金が発生し続ける（対策：RDS自動停止Lambda、`modules/rds-autostop`）
- ECSサービスを再作成するたびにタスク定義がbootstrapプレースホルダーイメージに戻るため、起動のたびにデプロイワークフローの再実行が必要
- 常時アクセスできないため、予告なしのデモ・共有には不向き（起動の一手間が必要）

## 関連情報
- コスト方針：[cost-high-level-spec.md](../../../../tomario-docs/basic-design/cost-high-level-spec.md)
- コスト試算の詳細：[cost-environment-design.md](../../../../tomario-docs/environment-definitions/cost-environment-design.md)
- RDS7日自動復旧への対策：方針は[cost-high-level-spec.md](../../../../tomario-docs/basic-design/cost-high-level-spec.md)、設定値（RDS自動停止Lambda）は[database-environment-design.md](../../../../tomario-docs/environment-definitions/database-environment-design.md)
