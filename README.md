# tomario-steering

Tomario（ホテル予約システム）の設計判断の記録（ADR）、非機能試験、リリース管理をまとめたリポジトリ。

## フォルダ構成

| フォルダ | 内容 |
|---------|------|
| [adr/](adr/) | ADR（Architecture Decision Record）。設計判断ごとに、背景・決定・選定理由・利点・欠点を記録する。`apps/`（アプリケーション）と`infra/`（インフラ、領域ごとのサブフォルダ）に分けている |
| [verification/non-functional-test/](verification/non-functional-test/) | 非機能試験の計画書・実施サマリ・試験種別ごとの手順書と結果報告書・エビデンス |
| [release-management/](release-management/) | 本番公開のリリース判定基準、カットオーバーと切り戻しの手順、公開直後の確認、ハイパーケアと引き継ぎ |

## 関連リポジトリ

| リポジトリ | 内容 |
|----------|------|
| [tomario-docs](https://github.com/Tomario-portfolio/tomario-docs) | 要件定義・基本設計・環境定義 |
| [tomario-infra](https://github.com/Tomario-portfolio/tomario-infra) | インフラのTerraformコードとCI/CD |
| [tomario-app](https://github.com/Tomario-portfolio/tomario-app) | アプリケーション（Flask）とフロントエンド、デプロイ用のCI/CD |
