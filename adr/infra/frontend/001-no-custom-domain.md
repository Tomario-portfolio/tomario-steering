# 001. 独自ドメイン・ACM証明書を導入しない

## ステータス
承認済み

## コンテキスト
フロントエンドはCloudFrontのデフォルトドメイン（`xxxx.cloudfront.net`）とデフォルト証明書（`*.cloudfront.net`）でHTTPSを提供している。独自ドメイン（Route 53）とACM証明書は、2026-07-11以降「検討中・後回し」としていた。

その後、非機能試験S-03（TLS設定の確認）で、TLS1.0/1.1が有効なままであることが判明した（cipherは`ECDHE-RSA-AES128-SHA`）。CloudFrontのデフォルト証明書では`minimum_protocol_version`を指定できないという仕様上の制約が原因で、TLS1.2以上に限定するには独自ドメインとACM証明書が必要になる。このため「独自ドメインを取得するときにまとめて対応する」としてスキップしていた（2026-09-27）。

残タスクの棚卸し（2026-10-04）にあたり、独自ドメインを取得するかどうかを最終的に決める必要があった。

## 決定
独自ドメイン・Route 53・ACM証明書は導入しない。CloudFrontのデフォルトドメインとデフォルト証明書での運用を続ける。

TLS1.0/1.1が有効なまま残ることは、既知のリスクとして受け入れる。

## 選定理由
- **導入する**：TLS1.2以上への限定、覚えやすいURLが得られる。一方で、ドメインの年額費用（$12〜15程度）、Hosted Zoneの月額（$0.50）、DNS・証明書の管理という固定費と運用の手間が発生する
- **導入しない**：本システムは非公開のポートフォリオ用途で、実ユーザーの通信を保護する必要性が低い。デフォルトドメインのままでもHTTPS自体は提供できており、機能上の不足は無い

費用と運用の手間に見合う必要性が無いと判断し、導入しない。

## 利点
- ドメイン・Hosted Zoneの固定費と、DNS・証明書の更新管理が不要
- 構成がシンプルなまま保たれる（CloudFront以外にDNS層を持たない）

## 欠点
- TLS1.0/1.1が有効なまま残り、判定基準「TLS1.2以上のみ」を満たさない（非機能試験S-03はスキップ扱い）
- URLが`xxxx.cloudfront.net`のままで、覚えにくい
- 一般公開して実ユーザーの通信を扱うことになった場合は、TLSの要件を満たすために再検討が必要。その場合でも既存リソースの作り直しは不要で、後から追加できる（Route 53・ACM（us-east-1）を追加し、CloudFrontの`viewer_certificate`を差し替える）

## 関連情報
- 設定値：[frontend-environment-design.md](../../../../tomario-docs/environment-definitions/frontend-environment-design.md)
- 非機能試験結果（S-03）：`tomario-steering/verification/non-functional-test/results/security-test-result.md`
