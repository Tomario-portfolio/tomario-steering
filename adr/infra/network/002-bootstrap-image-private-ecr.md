# 002. bootstrap_imageのプライベートECR参照化

## ステータス
承認済み

## コンテキスト
ECSサービスを新規作成・再作成する際（cost-start時を含む）、ECSタスク定義には何らかの初期イメージ（`var.bootstrap_image`）が必要になる。当初はパブリックのECR Gallery（`public.ecr.aws/docker/library/nginx:latest`）を参照していたが、本システムはNAT Gatewayを使わない構成（[ADR: NAT Gateway vs VPC Endpoint](001-vpc-endpoint-vs-nat-gateway.md)）のため、プライベートサブネットのECSタスクからパブリックエンドポイントへは到達できない。2026-07-10、cost-start後のサービス再作成時にこれが原因でクラッシュループが発生した。

## 決定
`bootstrap_image`をパブリックECR Galleryへの参照から、各環境のプライベートECR内に`bootstrap`タグで配置したプレースホルダーイメージへの参照に変更する。

## 選定理由
- **NAT Gatewayを導入する**：到達性の問題は解決するが、月$30〜45程度の固定費がかかり、NAT Gatewayを使わない設計方針（コスト最適化・攻撃面削減）そのものを覆すことになる
- **bootstrap_imageをプライベートECR参照に変更する**：既存のECR・VPCエンドポイント構成だけで解決でき、追加コストがゼロ。NAT Gatewayを使わない方針を維持できる

## 利点
- 追加コストゼロで解決できる
- NAT Gatewayを使わないというネットワーク設計の一貫性を保てる

## 欠点
- プレースホルダーイメージ自体を各環境のECRへ事前にpushしておく必要があり、新しい環境を作る際の手順が1つ増える

## 関連情報
- [network-high-level-spec.md](../../../../tomario-docs/basic-design/network-high-level-spec.md)・[compute-high-level-spec.md](../../../../tomario-docs/basic-design/compute-high-level-spec.md)
