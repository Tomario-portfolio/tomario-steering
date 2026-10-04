# 002. デプロイの安全網：デプロイサーキットブレーカー、Blue/Greenは見送り

## ステータス
承認済み（Blue/Greenは将来再検討）

## コンテキスト
デプロイ失敗時にサービスへ悪影響を与えないための安全網が必要だった。選択肢としてECSの標準機能である「デプロイサーキットブレーカー」と、CodeDeployによる「Blue/Greenデプロイ」があった。

## 決定
全環境共通でECSのデプロイサーキットブレーカー（`deployment_circuit_breaker { enable = true, rollback = true }`）を有効化する。Blue/Greenデプロイ（CodeDeploy）は導入を見送る。

## 選定理由
- **デプロイサーキットブレーカー**：ECSの標準機能のみで実現でき、追加のリソース（CodeDeployアプリケーション・デプロイメントグループ・テストリスナー等）が不要。デプロイ失敗時に自動的に直前の正常なリビジョンへロールバックする
- **Blue/Greenデプロイ**：より安全（新旧バージョンを並行稼働させてから切り替え）だが、デプロイ手順自体が変わる機能のため導入・検証コストが高い。production環境で初めて試すのはリスクが高く、staging環境で先に検証してから展開する必要がある

現時点ではローリングアップデート＋デプロイサーキットブレーカーで安全網を確保できていると判断し、Blue/Greenの導入は見送った。

## 利点
- 追加リソースなしで、デプロイ失敗時の自動ロールバックという安全網を全環境に適用できる
- 非機能試験（A-01）で実際にMTTR約1分50秒での自動ロールバックを確認済み

## 欠点
- デプロイサーキットブレーカーは「新旧完全並行稼働」までは行わないため、Blue/Greenほどの安全性（旧バージョンへの即時トラフィック切り戻し）は無い
- 導入する場合は改めてstagingでの検証が必要

## 関連情報
- [compute-high-level-spec.md](../../../../tomario-docs/basic-design/compute-high-level-spec.md)
- 非機能試験結果（A-01、MTTR約1分50秒）：`tomario-steering/verification/non-functional-test/results/availability-test-result.md`
