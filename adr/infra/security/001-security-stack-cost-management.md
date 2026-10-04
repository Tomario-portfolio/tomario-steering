# 001. セキュリティスタック（WAF・AWS Config・Security Hub）の「使う時だけ有効化」運用

## ステータス
承認済み

## コンテキスト
WAF・AWS Config・Security Hubはいずれも、非商用のdev/staging環境でCIS準拠チェック・Webアプリ層防御まで行う必要性は薄いが、production環境では実運用環境として一定の保護・可視化が欲しい。一方でこれら3サービスはいずれも「停止」という状態を持たず、「存在（課金）／削除・無効化（無課金）」の二択で、かつ時間按分課金（WAF Web ACLは常時起動で月$14〜16程度、Security Hub+Configは月$3〜5程度）である。production自体は一般公開前はdev/staging同様cost-stopで止めるが、このセキュリティスタックは常時稼働の有無とは別軸で「常時有効化しておくべきか」を判断する必要があった。

## 決定
WAF・AWS Config・Security Hubの3つをnonprod（dev/staging/shared）では無効のまま、productionでのみ導入する。かつproductionでも常時有効化はせず、`enable_security_stack`という1つのフラグでまとめてON/OFFし、検証や面接のタイミングなど必要な期間だけ有効化する運用とする。切り替えは`tomario-infra`の`security-stack.yml`（手動実行の専用ワークフロー）で行い、`cost-stop.yml`/`cost-start.yml`には組み込まない（cost-start/stop実行時もフラグの現在値をそのまま引き継ぎ、意図せずスタックが消えないようにするため）。

## 選定理由
- **常時有効化**：保護・可視化は最大化されるが、非公開のポートフォリオ期間中、実ユーザーがいない状態での常時課金（月$17〜21程度）に見合う価値が薄い
- **nonprodも含め全面導入**：検証環境の堅牢性は上がるが、非商用環境でCIS準拠チェックまで行う必要性が薄く、コストに見合わない
- **production限定・使う時だけ有効化**：実運用環境としての体裁（WAFでの防御・Security HubでのCIS準拠確認）は確保しつつ、非公開期間の課金をほぼゼロに抑えられる。cost-stop/startと同じ発想の応用

## 利点
- 非公開期間の追加コストをほぼゼロに抑えられる
- 1フラグでまとめて切り替えられるため、有効化し忘れ・無効化し忘れが起きにくい
- cost-stop/startとは独立したワークフローのため、通常のインフラ起動・停止操作でスタックが意図せず消えることがない

## 欠点
- 一般公開後、常時稼働に切り替えた場合でも「面接が近いタイミングだけ有効化する」運用を続ける想定のため、公開後の扱いは別途判断が必要（現状は未決定）
- WAFのCOUNT/BLOCKモード切り替えの仕組み自体は未実装で、常にBLOCKモードで入る（実ユーザーがいない間は誤検知の実害が無いため許容している。一般公開後に再検討）

## 関連情報
- 設計の詳細・コスト試算：[security-high-level-spec.md](../../../../tomario-docs/basic-design/security-high-level-spec.md)・[cost-high-level-spec.md](../../../../tomario-docs/basic-design/cost-high-level-spec.md)
- 導入経緯（当初は非商用のため見送っていたが、時間按分課金と判明し方針転換）：2026-07-11〜2026-08-03の判断
