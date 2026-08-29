# 001. NAT Gateway vs VPC Endpoint

## ステータス
承認済み

## コンテキスト
ECS FargateタスクはPrivate Subnetに配置し、外部インターネットに直接晒さない設計とする。一方でタスクはECR（イメージpull）・Secrets Manager（DB認証情報取得）・CloudWatch Logs（ログ出力）・S3（ECRレイヤー取得）といったAWSサービスへの到達性が必要で、Private SubnetからAWSサービスへの経路を用意する必要があった。

## 決定
NAT Gatewayを使わず、VPC Endpoint（Interface型4つ + Gateway型1つ）でAWSサービスに接続する構成を採用する。

## 選定理由
- **NAT Gateway**：インターネット経由でAWSサービスを含む任意の宛先に到達できる汎用的な経路だが、時間課金＋データ処理料金で月額約$32かかり、インターネットゲートウェイ経由という経路そのものが不要な露出になる
- **VPC Endpoint（Interface / Gateway型）**：必要なAWSサービスへの経路だけを個別に用意する。インターネットを経由せずAWSのプライベートネットワーク内で完結するため、コストと攻撃対象領域の両方を最小化できる

## 利点
- インターネットゲートウェイ／NAT Gatewayを経由しないため、Private Subnetのタスクが外部に一切露出しない
- Gateway型（S3）は無料。Interface型もNAT Gateway単体より合計コストを抑えられる
- cost-stop / cost-startでInterface型エンドポイントを削除・再作成でき、不使用時のコストをゼロにできる

## 欠点
- 必要なAWSサービスの数だけエンドポイントを個別に作成・管理する必要がある（NAT Gatewayなら1つで済むところを、今回は5つ管理している）
- 新しいAWSサービスへの依存が増えるたびに、対応するエンドポイントの追加を忘れずに行う必要がある

## 関連情報
- [Amazon VPC の料金](https://aws.amazon.com/vpc/pricing/)：NAT Gatewayの時間課金・データ処理料金、Gateway型VPCエンドポイントが無料であることの記載
- [AWS PrivateLink の料金](https://aws.amazon.com/privatelink/pricing/)：Interface型VPCエンドポイントの時間課金・データ処理料金の記載
