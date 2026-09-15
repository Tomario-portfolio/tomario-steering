# 非機能試験後に実施する残タスク

非機能試験(`non-functional-test/`)の実施中に見つかった、試験の完了を妨げないため後回しにした技術的な修正項目。

## CloudFrontのcustom_error_response(403→200)がWAFブロックまで200にマスキングする件

- 背景：S-07実施中に発覚。`modules/frontend/cloudfront.tf`の`custom_error_response`(403/404→200、SPAのクライアントサイドルーティング対応)がディストリビューション全体にかかるため、WAFがブロックして生成した403も200+index.htmlに上書きされる。WAF自体は正しく機能している(サンプリングログでは`BLOCK`と記録される)ため実害は無いが、ステータスコードだけでは「ブロックされたか」を判定できない
- 詳細・経緯：`tomario-workspace/task-and-flow/remaining-task.md` #18
- なぜ後回しか：WAFの防御機能自体には影響しない(見た目のステータスコードだけの問題)。`modules/frontend`は共有モジュールのため、直すとdev/staging/production全部に影響し、マージ時にdev/stagingの稼働中CloudFrontへ無承認でapplyされる（承認ゲートが無いため）。非機能試験の完了を優先し、試験終了後にまとめて対応する
- やること：SPAルーティングをCloudFront Function(viewer-request、拡張子の無いパスをindex.htmlへ書き換え)に置き換え、`custom_error_response`の403エントリを削除する。変更後、SPAのクライアントサイドルーティングが引き続き動くことを回帰確認する

## S-03：TLS1.0/1.1を無効化するには独自ドメイン取得が前提

- 背景：S-03(TLS設定の確認)で、TLS1.0/1.1が有効なまま(判定基準「TLS1.2以上のみ」未達)であることが判明
- 原因：`modules/frontend/cloudfront.tf`の`viewer_certificate`が`cloudfront_default_certificate = true`(独自ドメイン無し、`*.cloudfront.net`のデフォルト証明書)。CloudFrontのデフォルト証明書使用時は`minimum_protocol_version`を指定してTLSバージョンを絞ることができない仕様上の制約
- やること（独自ドメイン取得を決めた場合）：
  1. ドメイン取得（Route53で新規登録、または既存ドメインを使用）
  2. ACMで証明書発行（**us-east-1リージョン必須**、CloudFront用）、DNS検証
  3. `viewer_certificate`を`acm_certificate_arn` + `ssl_support_method = "sni-only"` + `minimum_protocol_version = "TLSv1.2_2021"`に変更
  4. `aliases`（カスタムドメイン）追加、Route53にCloudFrontへのエイリアスレコード作成
- 判断待ち：独自ドメイン取得の要否（年間$10〜15程度のコスト発生）。取得しない場合は、この制約をドキュメントに残して許容する
