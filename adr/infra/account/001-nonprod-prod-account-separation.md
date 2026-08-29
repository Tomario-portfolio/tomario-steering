# 001. アカウント戦略

## ステータス
承認済み

## コンテキスト
当初は単一のAWSアカウント内で`envs/dev`のみを運用していた。ここに、production同等構成で負荷試験・障害試験を行うstaging環境と、本番となるproduction環境を追加するにあたり、workload（dev/staging/production）をどうアカウントに配置するかを決める必要があった。

## 決定
nonprod（dev/staging/sharedを同居）とprod（production）の2アカウント構成に分離する。nonprod内はさらに、dev/stagingという2つのworkload環境と、両者が共有するshared環境（GuardDuty/CloudTrail/Budgets/ECR）に分ける。

## 選定理由（変遷）
当初は「1アカウント内でenv名やタグによって環境を分ける」案も比較検討した（3者対談形式で議論し、反論3点を検討した上で判断した）。

- GuardDuty・AWS Config・CloudTrail・AWS Budgetsはアカウント/リージョンにつき1つしか作成できないシングルトンリソースであり、同一アカウントに複数workload（dev/staging/production）が同居すると、これらのリソースの奪い合い・設定の使い回しが発生し、環境ごとの独立性が保てない
- productionは万一の設定ミスや誤操作の影響範囲を、他の検証環境から完全に隔離しておきたい。同一アカウントではIAM権限の設計だけで隔離しようとすると複雑になり、事故った時の影響範囲も同一アカウント内に留まる保証がない
- 一方でnonprod内のdev/stagingは、頻繁に環境ごと作り直す・検証するという性質上、sharedリソース（GuardDuty等）まで環境ごとに複製する必要性は薄く、dev/stagingの2workloadで1つのshared環境を共有する形にした

## 利点
- production環境の障害・誤操作が、nonprod側に一切影響しない
- GuardDuty等のシングルトンリソースを、workload環境の増減に関係なく安定して運用できる
- IAMロール・OIDC信頼条件もアカウント単位で完全に分離でき、最小権限の設計がシンプルになる

## 欠点
- AWSアカウントが増えることで、請求・IAM管理などの対象が増える
- nonprod内でdev/stagingがshared環境を共有するため、shared側の変更（例：GuardDuty設定変更）が両方に影響する
