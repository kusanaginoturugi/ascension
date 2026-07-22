# 聖明王院アカウント移行 見積・基礎資料

作成日: 2026-07-22

## 目的

`~/src/osystem` 配下の業務システム群を、現在の個人/既存アカウント前提から、聖明王院名義の Cloudflare / AWS / ドメイン / 認証基盤へ移すための棚卸し、移行方針、概算見積をまとめる。

この資料は初期見積用。実作業前に、現行 Cloudflare アカウントの D1/Workers/Pages 利用量、現行 EC2 またはサーバー上の production DB 実体、authentik の export 可否を確認して精度を上げる。

## 対象範囲

参照対象は `~/src/osystem` 配下の全ファイル。主なリポジトリは以下。

この資料では、取得する新ドメインの例を `myouou.net` とする。実際には、空きがあれば別のドメイン名を選べる。

| 区分 | ディレクトリ | 現行/想定スタック | 現行 URL / 備考 | 移行後 URL 案 |
| --- | --- | --- | --- | --- |
| ポータル | `portal/`, `WisdomKing/` | 静的 HTML/JS | GitHub repo は `WisdomKing` | `https://portal.myouou.net/` |
| 共通マスタ | `osystem-masters/` | Cloudflare Workers + D1 + Hono + authentik OIDC | `https://osystem-masters.kusanaginoturugi.workers.dev/` | `https://masters.myouou.net/` |
| 毎日集計 | `dailytally2/` | Cloudflare Workers + D1 + Browser Rendering + cron + authentik OIDC | `https://dailytally2.kusanaginoturugi.workers.dev/` | `https://dailytally2.myouou.net/` |
| 代理奉納 | `dedications/` | Rails 8.1 + Ruby 3.4.8 + SQLite + systemd | `https://dedications.showway.biz/` | `https://dedications.myouou.net/` |
| 超抜式 | `liberation/` | Rails 8.1 + Ruby 3.4.8 + SQLite | `https://liberation.showway.biz/` | `https://liberation.myouou.net/` |
| 道具販売 | `itementry/` | Rails 8.1 + Ruby 3.4.7 + SQLite | `https://itementry.showway.biz/` | `https://itementry.myouou.net/` |
| 道具一括注文 | `bulkpurchase/` | Rails 8.1 + Ruby 3.4.8 + SQLite + Solid Queue | `https://bulkpurchase.showway.biz/` | `https://bulkpurchase.myouou.net/` |
| テンキーレジ | `register/` | 静的 HTML + Python CLI | `https://register-xju.pages.dev/` | `https://register.myouou.net/` |
| 荷物番号 | `nimotsu-bango/` | 現状は静的検索ページ。README/親 README では Workers + D1 + OIDC 方針 | 実装状態と方針にズレあり | `https://nimotsu.myouou.net/` |
| 認証 | authentik | self-host | `https://auth.showway.biz/` | `https://auth.myouou.net/` |
| 一括メール案 | `bulksend/` | まだ資料段階。Cloudflare Email Service + Workers 案 | GitHub 作成未完了 | `https://bulksend.myouou.net/` |

## 現状から見える依存関係

### Cloudflare

- `dailytally2`
  - Worker: `dailytally2`
  - D1: `dailytally2`, database_id `ac0f9340-c703-40fd-b816-6063f16e2382`
  - Browser Rendering binding: `BROWSER`
  - Assets binding: `ASSETS`
  - cron: `*/15 * * * *`
  - public vars: `AUTHENTIK_ISSUER`, `AUTHENTIK_CLIENT_ID`, `REPORT_REMOTE_SUBMIT`, `TENDO_ACCOUNT`, `REPORT_NOTIFY_FROM`, `MASTERS_URL`
  - Secrets 再投入が必要: `AUTHENTIK_CLIENT_SECRET`, `SESSION_SECRET`, `TENDO_PASSWORD`, `RESEND_API_KEY` など
- `osystem-masters`
  - Worker: `osystem-masters`
  - D1: `osystem-masters`, database_id `2a5b1ce7-e309-4291-b28e-34fbada65526`
  - public vars: `AUTHENTIK_ISSUER`, `AUTHENTIK_CLIENT_ID`
  - Secrets 再投入が必要: OIDC client secret, session secret 系
- `.old/Dailytally`
  - 旧 Worker/D1: `dailytally`, database_id `0ebc08db-cb6b-466a-9e0f-a6904f6626ed`
  - アーカイブ扱い。移行対象は基本 `dailytally2` だが、監査用バックアップは保持する。
- `register`
  - 現行は Pages URL。独自ドメイン配下へ移すなら Cloudflare Pages か静的 Worker へ移す。
- `bulksend`
  - 未実装。Cloudflare Email Service + Workers + D1 で作る方針資料あり。

### AWS / EC2 / Lightsail / Rails

Rails 系 4 本はすべて SQLite の production DB を `storage/production.sqlite3` に置く前提。`dedications`, `itementry`, `bulkpurchase` には `/home/admin/<app>` 前提の `scripts/deploy.sh` があり、systemd サービスを restart する作り。

| アプリ | production ポート | 現行デプロイ手がかり | 永続化 |
| --- | ---: | --- | --- |
| `dedications` | 3003 | `scripts/deploy.sh`, `config/deploy.yml` 雛形 | `storage/` |
| `itementry` | 3000 | `scripts/deploy.sh`, `config/deploy.yml` 雛形 | `storage/` |
| `liberation` | 3000 | README は `mise` 前提。deploy script は未確認 | `storage/` |
| `bulkpurchase` | 3005 | `scripts/deploy.sh`, README に EC2 Ubuntu 24.04 手順 | `storage/` |

同一 EC2 に載せる場合、Puma ポート衝突を整理する。`itementry` と `liberation` はどちらも既定 3000 なので、systemd の `PORT=` で分ける。

現行サーバーが `t4g` の場合は ARM64。Lightsail に移す場合は amd64 前提で見る。systemd-nspawn の ARM64 rootfs をそのまま持っていくのではなく、Debian amd64 rootfs を作り直し、アプリコード、SQLite DB、systemd unit、nspawn 設定を移植する。元々 t4a/amd64 系から来ているなら、作業は増えるが構造的な問題は小さい。

### authentik

現行参照先は `https://auth.showway.biz/`。Workers 側は OIDC Provider の issuer URL と client ID が `wrangler.toml` に入っている。移行時は新しい authentik で Provider/Application を作り直し、以下をすべて差し替える。

- `AUTHENTIK_ISSUER`
- `AUTHENTIK_CLIENT_ID`
- `AUTHENTIK_CLIENT_SECRET`
- Redirect URI: `https://<移行後ホスト>/auth/callback`
- group mapping: admin / dailytally-admin / 各伝道会名
- Forward Auth を使う Rails アプリがある場合は Cloudflare/Nginx 側設定も移植

authentik 自体は Cloudflare Workers には向かない。PostgreSQL/Redis/worker プロセスが必要なので、現実案は以下。

1. Lightsail または EC2 に authentik を同居
2. authentik だけマネージド VPS/専用 EC2 へ分離
3. 将来、利用者数や可用性要件が上がったら RDS + ElastiCache へ分離

初期移行では 1 が一番安くて速い。今回の規模では、まず同居構成で始める。

## 推奨移行アーキテクチャ

### 推奨案 A: 小さく始める

- Cloudflare
  - 聖明王院アカウント作成
  - 独自ドメイン取得
  - DNS を Cloudflare 管理
  - Workers Paid を有効化
  - `osystem-masters`, `dailytally2` を新アカウントへ再作成
  - `register`, `portal` は Pages か静的 Worker
  - `nimotsu-bango` はいったん静的公開。更新・OIDC 必須化は別工程
- AWS
  - 聖明王院アカウント作成
  - EC2 1 台 + EBS + Elastic IP
  - Rails 4 本 + authentik を同居
  - Nginx/Caddy でリバースプロキシ
  - Cloudflare DNS から各サブドメインを EC2 へ
  - SQLite は EBS 上でバックアップ

この案の利点は初期費用と作業量が少ないこと。弱点は EC2 1 台に障害点が寄ること。

### 推奨案 B: 少し堅め

- Cloudflare は案 A と同じ
- AWS
  - EC2 2 台
    - app: Rails 4 本
    - auth: authentik
  - authentik の PostgreSQL を RDS にするか、少なくとも EBS snapshot + DB dump を別管理
  - Rails SQLite はアプリごとの EBS backup

可用性と復旧性は上がるが、月額と運用手順は増える。今の規模なら、まず案 A で移し、バックアップ/監視をきちんと組む方が現実的。

### 推奨案 C: Lightsail で月額を下げる

- Cloudflare は案 A と同じ
- AWS
  - Lightsail 4GB RAM プラン
  - 2 vCPU、80GB SSD、4TB/月 転送量込み
  - Debian amd64
  - static IP
  - systemd-nspawn を継続
  - Rails 4 本 + authentik を同居
  - SQLite / authentik DB はホスト側ディレクトリに bind mount
  - Lightsail snapshot + SQLite `.backup` の二段構え

この案は月額がわかりやすく、EC2 より安い。Lightsail が安い理由は、小規模サーバー向けの月額パックとして用途を絞っているため。サーバー、SSD、転送量がまとまっていて扱いやすい一方、EC2 のように複数台構成、RDS、ロードバランサ、高度なネットワーク設計へ広げる自由度は低い。

メリット:

- 月額が安く、決裁説明がしやすい
- サーバー、SSD、転送量がパックになっていて見積が読みやすい
- 単一サーバーで Rails 4 本 + authentik を動かす構成に合う
- AWS アカウント内で完結できる

デメリット:

- EC2 より規模拡張の自由度が低い
- RDS、ロードバランサ、複数台構成へ広げるなら EC2 のほうが自然
- バックアップや監視の選択肢は EC2 より少ない
- 現行 `t4g` ARM64 から amd64 Debian rootfs を作り直す分の移行作業が増える

50名規模で単一サーバー運用なら、費用面では第一候補にできる。将来、本格的な冗長化やDB分離が必要になった時点で EC2 へ移す。

Lightsail 単体でできる拡張:

- 上位プランへ移す
  - 4GB RAM / 2 vCPU / 80GB SSD から、8GB RAM / 2 vCPU / 160GB SSD などへ上げられる
  - 手順は snapshot を取り、より大きいプランで新インスタンスを作る流れ
  - そのため、厳密には「その場でCPU/メモリだけ変更」ではなく、作り直しに近い
- ディスク容量を増やす
  - 追加 block storage を attach できる
  - SQLite、authentik DB、バックアップ置き場を追加ディスクへ逃がせる
- 静的IPを付け替える
  - 新しい Lightsail インスタンスへ static IP を付け替えれば、Cloudflare DNS 側の変更を小さくできる
- snapshot から復旧する
  - インスタンス snapshot を使って、同じ構成のサーバーを作り直せる
  - ただし SQLite / PostgreSQL の整合性は、アプリ側の `.backup` や DB dump と組み合わせるほうが安全

Lightsail 単体で弱い拡張:

- 複数台に分けて自動負荷分散する構成
- DB をマネージドサービスとして分離する構成
- 細かい VPC / Security Group / IAM / AWS Backup 設計
- 高度な監視、ログ集約、障害時の自動復旧

今回の運用では、Rails アプリは大きく増やさず、軽い新規アプリは Cloudflare Workers に載せる前提。そのため、Lightsail は「既存 Rails 系 + authentik の常駐基盤」として使い、増える部分は Cloudflare 側に逃がせる。Lightsail 4GB で始め、足りなければ 8GB へ上げる、さらに本格的な分離が必要になったら EC2 へ移す、という段階的な判断でよい。

## ドメイン設計案

仮に `myouou.net` または同等の独自ドメインを取得する前提。`myouou.net` は例であり、空きがあれば別のドメイン名を選べる。Cloudflare Registrar で取得できるドメインなら、そのまま Cloudflare DNS / Workers / Email Service と組み合わせやすい。

| ホスト | 移行先 |
| --- | --- |
| `portal.<domain>` | ポータル |
| `masters.<domain>` | `osystem-masters` |
| `dailytally2.<domain>` | `dailytally2` |
| `dedications.<domain>` | Rails / EC2 |
| `liberation.<domain>` | Rails / EC2 |
| `itementry.<domain>` | Rails / EC2 |
| `bulkpurchase.<domain>` | Rails / EC2 |
| `register.<domain>` | Pages/静的 Worker |
| `nimotsu.<domain>` | 静的 Worker または Workers+D1 |
| `auth.<domain>` | authentik |
| `mail.<domain>` / `notice@<domain>` | Cloudflare Email Routing / Email Service 検討 |

`showway.biz` からの移行時は、いきなり切り替えず、先に新ドメインで並行稼働し、ログイン・帳票・外部送信を確認してから旧ドメインを案内停止する。

## データ移行方針

### Cloudflare D1

1. 現行アカウントで D1 export/backup
2. 新アカウントで D1 create
3. `wrangler.toml` の `database_id` を新 ID に変更
4. migrations 適用
5. export SQL/JSON を投入
6. Worker secrets を再投入
7. 新ドメインで `/api/bootstrap`, `/api/me`, `/auth/login`, `/auth/callback` を確認

注意点:

- D1 database_id はアカウント間で引き継げない。必ず新規作成。
- `MASTERS_URL` は新 `masters.<domain>` へ変更する。
- `dailytally2` は Browser Rendering と cron を使うため、Workers Paid 前提で見積もる。

### Rails SQLite

1. 現行サーバーで各アプリを一時停止またはメンテナンス
2. `sqlite3 storage/production.sqlite3 ".backup 'backup.sqlite3'"`
3. `storage/production_cache.sqlite3`, `production_queue.sqlite3`, `production_cable.sqlite3` は必要に応じて再生成。業務データは主に `production.sqlite3`
4. 新 EC2 へコード配置
5. `RAILS_MASTER_KEY` / `config/master.key` を安全に配置
6. backup SQLite を `storage/production.sqlite3` に配置
7. `bin/rails db:migrate`
8. `assets:precompile`
9. systemd 起動
10. `/up` と主要画面を確認

注意点:

- `config/master.key` がローカルに存在するリポジトリがある。Git で扱わず、移行作業では Secrets として別経路で渡す。
- Rails 個別 users と authentik 統合は別タスク。移行時に同時実施するとリスクが上がる。
- `dedications`, `itementry`, `bulkpurchase` の deploy script は `git reset --hard` を含む。新環境用にリポジトリ状態と手順を整えるまで手動実行は慎重にやる。

### systemd-nspawn / Lightsail

Lightsail 案では、現行 ARM64 rootfs の丸ごと移行ではなく、amd64 rootfs の再作成として扱う。

1. Lightsail に Debian を作成
2. static IP を割り当て
3. `systemd-container`, `machinectl`, `systemd-nspawn` 周辺をセットアップ
4. Debian amd64 rootfs を作成
5. 現行 nspawn 設定、systemd unit、bind mount 設定を移植
6. Ruby / Node / SQLite / system packages を amd64 側で入れ直す
7. Rails アプリコードと `storage/production.sqlite3` を配置
8. authentik の PostgreSQL/Redis/設定データを復元または再作成
9. Cloudflare DNS を Lightsail static IP へ向ける

注意点:

- ARM64 native build 済みの gem/node module は流用しない。amd64 側で入れ直す。
- SQLite DB ファイル自体はアーキテクチャ非依存なので移せる。
- バックアップ対象は rootfs 全体ではなく、ホスト側の `/opt/osystem/data` のような永続ディレクトリに寄せる。

### authentik

現行 authentik の移行方法は未確認。選択肢は以下。

- 推奨: 設定 export/import または Terraform 相当で Application/Provider/Group を再作成
- 次善: 管理画面から手動再作成
- ユーザー移行: パスワードを持ち出せない/持ち出さない前提なら、初回パスワード再設定運用にする

OIDC client secret は新規発行する。既存 secret をそのまま使う前提にしない。

## 作業見積

前提:

- GitHub リポジトリは現行のまま使うか、聖明王院 organization へ移すかは未決。
- 実作業者 1 名、レビュー/確認者 1 名。
- 業務データの最新バックアップ取得と最終切替は、利用者が少ない時間帯に行う。

| フェーズ | 内容 | 概算工数 |
| --- | --- | ---: |
| 1. 詳細棚卸し | 現行 Cloudflare/AWS/authentik/GitHub/ドメイン/Secrets/DB サイズ/利用量の確認 | 1.0-1.5 日 |
| 2. 新アカウント準備 | Cloudflare, AWS, billing, MFA, IAM, ドメイン, DNS 基本設定 | 0.5-1.0 日 |
| 3. Cloudflare 移行 | D1 作成/投入、Workers deploy、Pages/静的移行、cron/Browser/Secrets、独自ドメイン | 1.5-2.5 日 |
| 4. AWS/EC2 構築 | VPC/SG/EC2/EBS/Elastic IP、OS hardening、Docker or Ruby/mise、Nginx/Caddy、systemd | 1.0-2.0 日 |
| 5. Rails 移行 | Rails 4 本の配置、SQLite 移行、master key、assets、service、動作確認 | 2.0-3.0 日 |
| 6. authentik 移行 | 新環境構築、Provider/Application/Group、OIDC callback、ユーザー再設定 | 1.0-2.0 日 |
| 7. 結合確認 | ログイン、マスタ同期、帳票、PDF、tendo.net 送信、メール通知、バックアップ復旧訓練 | 1.5-2.5 日 |
| 8. 切替/引き継ぎ | DNS 切替、旧環境凍結、運用手順、障害時手順、管理者説明 | 1.0 日 |

合計:

- 小さく始める案 A: 9-13 人日
- Lightsail 案 C: 9.5-14 人日
- 堅め案 B: 12-17 人日
- 追加で GitHub organization 移管、Rails の authentik 統合、`nimotsu-bango` の Workers+D1 化、`bulksend` 実装まで含める場合: +5-12 人日

Lightsail 案は AWS 構築が少し軽くなる一方で、ARM64 から amd64 rootfs へ戻す作業が増える。差し引きで、EC2 案より **+0.5〜1.0日** 見ておく。

## 月額費用の初期目安

2026-07-22 時点で確認した公開価格ベース。税、為替、リージョン差、データ転送、メール送信量、ログ保存量は別。

| 項目 | 目安 |
| --- | ---: |
| Cloudflare Workers Paid | $5/月から。10M requests/月、30M CPU ms/月込み。超過は requests $0.30/M、CPU $0.02/M CPU ms |
| Cloudflare D1 | Free/Paid とも 5GB まで込み。Paid は 25B rows read/月、50M rows written/月込み。超過は read $0.001/M rows、write $1.00/M rows、storage $0.75/GB-month |
| Cloudflare Registrar | 原価販売。TLD による。Cloudflare の公開ページでは “starting at $7.85” 表示あり |
| EC2 t4g.medium | Rails 4 本 + authentik 同居の比較対象 |
| Lightsail 4GB | 小規模サーバー向けの月額パック。2 vCPU、80GB SSD、4TB/月 転送量込み。今回の費用優先案 |
| EBS gp3 | us-east-1 例 $0.08/GB-month。50GB なら約 $4/月、100GB なら約 $8/月 |
| Elastic IP | 実運用では課金対象になり得る。AWS の最新請求で確認 |
| Snapshot/S3 backup | 容量次第。SQLite の現状規模なら当面小さいが、世代数で増える |

ざっくり:

- 案 A 最小: Cloudflare $5/月 + EC2/EBS 約 $30-60/月 + ドメイン年額
- 案 A 余裕あり: Cloudflare $5-20/月 + EC2/EBS 約 $60-120/月 + ドメイン年額
- 案 B: EC2/RDS 等を分けるため $100/月超を見込む

## 実行チェックリスト

### アカウント/権限

- Cloudflare 聖明王院アカウントを作成
- Cloudflare 管理者 2 名以上、MFA 必須
- AWS 聖明王院アカウントを作成
- AWS root MFA、管理 IAM Identity Center または IAM user 最小権限
- GitHub owner / deploy key / Actions secrets の移管方針を決める
- ドメイン候補を決定し、取得

### Cloudflare

- Workers Paid 有効化
- `osystem-masters` D1 作成、migration、seed/import
- `dailytally2` D1 作成、migration、import
- `wrangler.toml` の account-specific 値を差し替え
- Secrets 再投入
- `masters.<domain>`, `dailytally2.<domain>` routes/custom domains 設定
- Browser Rendering binding 確認
- cron trigger 確認
- Observability/logs の保持方針決定

### AWS/EC2

- リージョン決定。日本の利用者中心なら ap-northeast-1、安さ優先なら us-east-1 も検討
- EC2 instance type 決定
- EBS サイズ、snapshot 世代、backup 保存先決定
- Security Group: 22 は管理元限定、80/443 は Cloudflare からのみも検討
- OS: Ubuntu 24.04
- reverse proxy: Caddy または Nginx
- systemd unit を 4 Rails + authentik で整備
- logrotate / journald retention / disk alert

### AWS/Lightsail

- Lightsail 4GB RAM プランを選ぶ。2 vCPU、80GB SSD、4TB/月 転送量込み
- Debian amd64 を選ぶ
- static IP を割り当てる
- systemd-nspawn をセットアップ
- amd64 rootfs を作り直す
- ARM64 由来の gem/node module を持ち込まず、移行先で入れ直す
- SQLite DB と Active Storage はホスト側永続ディレクトリに置く
- Lightsail snapshot と SQLite `.backup` の定期実行を設定する

### authentik

- `auth.<domain>` を用意
- PostgreSQL/Redis を含む deployment 方法を決定
- Application/Provider を再作成
- groups を再作成
- 管理者/利用者の初期ログイン手順を作る
- OIDC callback を新 URL に合わせる

### 切替

- 新環境でデータ投入
- 新 URL で全アプリ smoke test
- 管理者ログイン確認
- マスタ同期確認
- PDF/CSV 出力確認
- tendo.net 送信はテストまたは明示許可後に本番確認
- メール通知 From/Reply-To/SPF/DKIM/DMARC 確認
- 旧環境を read-only または案内停止
- DNS 切替
- 旧環境バックアップを保存

## 未確認事項

- 新ドメイン名
- Cloudflare アカウントを誰が owner にするか
- GitHub リポジトリを `kusanaginoturugi` から聖明王院 organization へ移すか
- 現行 Cloudflare の D1 最新データ取得権限
- 現行 `showway.biz` サーバーの実体、systemd unit、Nginx 設定、production DB 容量
- 現行 authentik の export/backup 方法
- Rails 4 本を今後も個別ログインで残すか、authentik に統合するか
- `nimotsu-bango` を静的のまま移すか、Workers+D1+OIDC 実装まで今回に含めるか
- `bulksend` を今回の移行に含めるか
- メール送信元ドメインと返信先
- バックアップ保持期間、復旧目標時間

## 参照したローカル資料

- `/home/onoue/src/osystem/README.md`
- `/home/onoue/src/osystem/DATA.md`
- `/home/onoue/src/osystem/dailytally2/README.md`
- `/home/onoue/src/osystem/dailytally2/wrangler.toml`
- `/home/onoue/src/osystem/osystem-masters/README.md`
- `/home/onoue/src/osystem/osystem-masters/wrangler.toml`
- `/home/onoue/src/osystem/dedications/README.md`
- `/home/onoue/src/osystem/dedications/scripts/deploy.sh`
- `/home/onoue/src/osystem/liberation/README.md`
- `/home/onoue/src/osystem/itementry/README.md`
- `/home/onoue/src/osystem/itementry/scripts/deploy.sh`
- `/home/onoue/src/osystem/bulkpurchase/README.md`
- `/home/onoue/src/osystem/bulkpurchase/scripts/deploy.sh`
- `/home/onoue/src/osystem/register/README.md`
- `/home/onoue/src/osystem/nimotsu-bango/README.md`
- `/home/onoue/src/osystem/bulksend/docs/*.md`

## 参照した外部資料

- Cloudflare Registrar: https://developers.cloudflare.com/registrar/
- Cloudflare Workers pricing: https://developers.cloudflare.com/workers/platform/pricing/
- Cloudflare D1 pricing: https://developers.cloudflare.com/d1/platform/pricing/
- Cloudflare plans/pricing: https://www.cloudflare.com/plans/
- AWS pricing overview: https://aws.amazon.com/pricing/
- AWS EC2 On-Demand: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-on-demand-instances.html
- AWS EBS pricing: https://aws.amazon.com/ebs/pricing/
- AWS EBS gp3 overview: https://aws.amazon.com/ebs/general-purpose/

## 作業記録

- 2026-07-22: `~/src/osystem` 配下の README、wrangler、Rails database/storage/puma/routes/deploy script、既存 docs を確認。
- 2026-07-22: Cloudflare/AWS の公開価格ページを確認し、初期見積へ反映。
- 2026-07-22: 本資料を作成。

## 引き継ぎ

次にやること:

1. 現行 Cloudflare にログインして D1 export、Workers routes、Pages、Secrets 名、利用量を確認する。
2. 現行サーバーにログインして Rails 4 本の production DB、systemd unit、reverse proxy 設定、backup 状況を取得する。
3. 現行 authentik の backup/export 方針を確認する。
4. ドメイン名と GitHub organization 移管方針を決める。
5. 案 A / 案 B のどちらで進めるか決め、詳細手順書に落とす。
