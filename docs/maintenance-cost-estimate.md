# 聖明王院システム 維持費見積

作成日: 2026-07-22

## 結論

毎月の維持費は、AWS EC2 なら **約 7,800円/月**、AWS Lightsail なら **約 5,500円/月** が目安。

安さとわかりやすさを優先するなら **Lightsail**。AWS の本格構成へ広げやすくするなら **EC2**。

ここでは、わかりやすさ優先で次の条件に固定している。

- 為替: `1ドル = 163円`
- 表示: 月額のみ
- EC2案: AWS EC2 `t4g.medium` + 32GB ストレージ
- Lightsail案: AWS Lightsail 4GB RAM プラン
- 認証: authentik は self-host のみ
- ドメイン: `myouou.net` を Cloudflare で取得して継続使用
- Cloudflare Workers と Email Service を使用
- 利用者: 50名

## 月額見積

| 案 | 月額目安 | 説明 |
| --- | ---: | --- |
| Lightsail 4GB | **約 5,500円/月** | 月額固定でわかりやすい。今回の第一候補。 |
| EC2 `t4g.medium` 毎月払い | **約 7,800円/月** | いつでもやめやすい。本格AWS構成へ広げやすい。 |
| EC2 `t4g.medium` 1年契約割引 | **約 6,400円/月** | 1年は使い続ける前提。EC2毎月払いより安い。 |
| EC2 `t4g.medium` 3年契約割引 | **約 5,500円/月** | EC2では一番安いが、3年縛りなので最初からはおすすめしない。 |

おすすめは、**Lightsail 4GB で始める**。EC2を選ぶ場合は、最初の1-2か月は毎月払いで実際の負荷を見て、その後に1年契約割引へ切り替える。

## 内訳

Lightsail 4GB の場合:

| 項目 | 月額目安 | 内容 |
| --- | ---: | --- |
| Cloudflare | 約 980円 | Workers Paid、メール送信、DNS、ドメイン月割り |
| AWS Lightsail | 約 3,920円 | 4GB RAM、2 vCPU、SSD込みの月額プラン |
| 予備 | 約 600円 | 端数、軽い超過、ログなどの余裕 |
| 合計 | **約 5,500円/月** |  |

EC2 毎月払いの場合:

| 項目 | 月額目安 | 内容 |
| --- | ---: | --- |
| Cloudflare | 約 980円 | Workers Paid、メール送信、DNS、ドメイン月割り |
| AWS EC2 | 約 5,140円 | `t4g.medium` サーバー 1台 |
| AWS ストレージ | 約 500円 | 32GB |
| AWS IPv4 | 約 600円 | サーバー公開用 IP アドレス |
| 予備 | 約 600円 | 端数、軽い超過、ログなどの余裕 |
| 合計 | **約 7,800円/月** |  |

EC2 1年契約割引の場合:

| 項目 | 月額目安 | 内容 |
| --- | ---: | --- |
| Cloudflare | 約 980円 | Workers Paid、メール送信、DNS、ドメイン月割り |
| AWS EC2 | 約 3,240円 | `t4g.medium` サーバー 1台、1年契約割引 |
| AWS ストレージ | 約 500円 | 32GB |
| AWS IPv4 | 約 600円 | サーバー公開用 IP アドレス |
| 予備 | 約 1,100円 | 端数、軽い超過、ログなどの余裕 |
| 合計 | **約 6,400円/月** |  |

## Lightsail のメリット・デメリット

Lightsail が EC2 より安い理由は、用途を小規模サーバー向けに絞った月額パックだから。サーバー、SSD、転送量がまとまっていて、料金が読みやすい。その代わり、EC2 のように細かく部品を組み合わせて大きな構成へ広げる自由度は低い。

| 観点 | Lightsail | EC2 |
| --- | --- | --- |
| 料金 | 安く、月額がわかりやすい | 高めだが細かく調整できる |
| 初期構築 | シンプル | ネットワーク、ディスク、IPなどを個別に設計する |
| 小規模運用 | 向いている | 使えるがやや大げさ |
| 将来の拡張 | 弱い。単一サーバー運用向き | 強い。複数台、RDS、ロードバランサへ広げやすい |
| バックアップ | Lightsail snapshot中心で簡単 | EBS snapshot、AWS Backupなど選択肢が多い |
| 判断 | 今回の第一候補 | 将来の本格拡張を重視する場合 |

今回の規模は利用者50名程度なので、まず Lightsail で始める判断は自然。将来、利用者やアクセスが大きく増えたり、DBを分離したり、複数台構成にしたくなったら EC2 へ移す。

## 何が含まれるか

この見積に含めるもの:

- 業務アプリを動かす AWS サーバー
- Rails 系アプリ
- authentik self-host
- Cloudflare Workers
- Cloudflare D1
- Cloudflare Email Service
- `myouou.net` のドメイン維持費
- DNS
- サーバー公開用 IP アドレス
- EC2案では 32GB ストレージ
- Lightsail案ではプラン付属SSD

この見積に含めないもの:

- 作業費
- 消費税
- 為替変動
- 大量メール送信の超過
- 大量アクセス時の超過
- バックアップを大きく増やした場合の追加容量
- 有償サポート契約

## メール料金

Cloudflare Email Service は、Cloudflare Workers Paid に **3,000通/月** まで含まれる。

50名利用なら、通常の連絡メールや通知メールはこの範囲で足りる見込み。

超えた場合の追加料金:

| 送信数 | 追加料金目安 |
| ---: | ---: |
| 3,000通/月まで | 0円 |
| 10,000通/月 | 約 400円/月 追加 |
| 20,000通/月 | 約 970円/月 追加 |

## 補足

authentik は self-host に限定する。以前比較した authentik Enterprise は 50名で月額約4万円を超えるため、今回の規模では使わない前提にする。

Lightsail は EC2 `t4g` と違い、移行先が amd64 になる前提で見る。現状の ARM64 rootfs をそのままコピーして起動するのではなく、Debian amd64 環境を作り直し、アプリと SQLite データを移す。

## 参照

- Cloudflare Workers pricing: https://developers.cloudflare.com/workers/platform/pricing/
- Cloudflare Email Service pricing: https://developers.cloudflare.com/email-service/platform/pricing/
- Cloudflare Browser Run pricing: https://developers.cloudflare.com/browser-run/pricing/
- Cloudflare Registrar renewal: https://developers.cloudflare.com/registrar/account-options/renew-domains/
- AWS EC2 On-Demand pricing: https://aws.amazon.com/ec2/pricing/on-demand/
- AWS Lightsail pricing: https://aws.amazon.com/lightsail/pricing/
- AWS Savings Plans overview: https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html
- AWS EBS pricing: https://aws.amazon.com/ebs/pricing/
- AWS public IPv4 charge: https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/
