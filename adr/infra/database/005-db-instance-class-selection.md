# 005. RDSインスタンスクラスの選定

## ステータス
承認済み

## コンテキスト
ホテル予約システムのDB負荷は、非公開のポートフォリオ用途では小さく、本番相当の性能を常時確保する必要性は薄い。一方でstagingでの負荷試験時だけは、Auto Scalingの実挙動検証のためある程度の負荷をRDSにもかけたい。全環境で同一のインスタンスクラスにするか、環境ごとに変えるかを決める必要があった。

## 決定
全環境（dev/staging/production）で`db.t3.micro`を基本のインスタンスクラスとする。stagingでの負荷テスト直前のみ、AWS CLIで一時的に`db.t4g.medium`（Graviton、固定4GiBメモリ）にスケールアップし、終了後に`db.t3.micro`へ戻す。Terraform上の値（`db.t3.micro`）は変更しない。

## 選定理由
- **全環境を常時大きめのインスタンスクラスにする**：性能的な余裕は生まれるが、非公開期間のコストが不必要に増える。垂直スケールをしない（水平スケールで対応する）という全体方針にも反する
- **全環境`db.t3.micro`固定＋負荷テスト時のみ一時変更**：平常時のコストを抑えつつ、負荷テスト時だけAWS CLIで`apply_immediately`によるスケールアップ/ダウンを行うことで、Terraformの恒久設定を変えずに性能検証ができる。`t4g.medium`を選んだのは、Gravitonの固定4GiBメモリによりCPUクレジット枯渇の心配が少なく、負荷テスト結果が安定するため

## 利点
- 平常時のRDSコストを最小限に抑えられる（`db.t3.micro`は無料枠対象にもなりうる低価格帯）
- 負荷テストのたびに恒久的なインフラ変更（Terraform apply）が不要で、CLIコマンド一発でスケール変更・復元ができる

## 欠点
- `db.t3.micro`等の小さいインスタンスクラスでは、RDS Performance Insightsがサポートされない（[ADR: RDS Performance Insightsの導入見送り](004-performance-insights-instance-class-limitation.md)参照）
- 一時的なスケールアップ/ダウンの手順が手動（AWS CLI）であり、実行し忘れる・戻し忘れるリスクがある

## 関連情報
- [database-high-level-spec.md](../../../../tomario-docs/basic-design/database-high-level-spec.md)
