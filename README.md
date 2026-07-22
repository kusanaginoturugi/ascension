# ascension

聖明王院向け業務システム群を、聖明王院名義の Cloudflare / AWS / ドメインへ移行するための検討資料置き場。

## 資料

| ファイル | 内容 |
| --- | --- |
| [docs/system-migration-assessment.md](docs/system-migration-assessment.md) | システム全体の棚卸し、移行方針、工数見積、実行チェックリスト。まず全体像を掴む資料。 |
| [docs/maintenance-cost-estimate.md](docs/maintenance-cost-estimate.md) | ドメイン、Cloudflare、AWS EC2 / Lightsail、メールサービスを含めた月額維持費の見積。決裁説明向け。 |

## 読み方

1. まず [system-migration-assessment.md](docs/system-migration-assessment.md) で、移行対象と作業範囲を見る。
2. 次に [maintenance-cost-estimate.md](docs/maintenance-cost-estimate.md) で、毎月いくらかかるかを見る。

## 現時点の整理

- Cloudflare Workers / D1 系は、新しい Cloudflare アカウントで作り直してデータを移す。
- Rails 系と authentik は、AWS 上のサーバーに self-host する。
- 費用優先なら Lightsail 4GB が第一候補。
- EC2 を選ぶ場合は `t4g.medium` を基準にする。
- Lightsail は安くて月額が読みやすい代わりに、EC2 より大規模拡張の自由度は低い。

## 注意

親ディレクトリ `/home/onoue/src/osystem` は、各サブディレクトリを独立リポジトリとして扱う構成。親リポジトリの `.gitignore` はトップレベル資料だけを追跡する allowlist なので、この `ascension/` 配下の資料を git 管理する場合は扱いを別途決める。
