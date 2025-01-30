---
title: "CloudFlareのプロキシを利用してFirebaseHostingしたサービスのTLS1.2暗号スイートを制限する"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [network, Security, Firebase, cloudflare]
published: true
---

## モチベーション

Firebase Hosting にデプロイしたサービスに対して、SSL LabsでスキャンをかけたところTLS1.2の中に脆弱な暗号スイート（WEAK）が発見されました。別の記事では、WEAKの存在がすぐさまクリティカルなセキュリティインシデントになるかどうかはケースバイケースと書きました。

このたび、連携先サービスからなんとしてもWEAKを無くせ（意訳）と指摘をうけたので、CloudFlareを利用して対応しました。

意外とノウハウ記事が少なかったのでまとめておきます。

## ゴール

- Firebase Hosting にデプロイしているサイトへのアクセスにおいて古い暗号スイートの利用をやめる[^1]


[^1]: 正確に言うと、システム全体としてアクセスを制御する、ですね

### 補足

- ちょっとしたAPI機能があるため`Firebase Hosting + Functions` の構成。構成変更はコストが高く避けたい。
- ドメイン管理・DNSの設定は `CloudFlare` を利用している
- サブドメインなど、いくつか環境が存在するが、すべて同じ設定変更で良い
- Firebase Hosting では個別に暗号スイートをカスタマイズできない
  - Googleが自動的に用意するためカスタマイズできない
  - TLS1.2以上の通信を強制、TLS1.3もサポート
  - TLS1.2は後方互換性のためか、2025年1月時点ではWEAKな暗号スイートが含まれている（cbc系の暗号スイート）
  - ただし、TLS1.2の暗号スイートに優先順位がつけられており、優先度の高いものから利用されるため基本問題ない
- 安全でないTLS1.2の暗号スイートを利用するクライアントは相当古い
  - cbc系の暗号スイートで通信をするクライアントは相当古く現代で利用している人はほぼいない…気がする

## 実装のまとめ（結論）

- Cloudflareのゾーンに対してAPI経由でTLS1.2の暗号スイートを変更（変更はAPI経由のみ可能）
- DNS設定に対してキャッシュを有効化し、クライアントからのアクセスをCloudflare経由に変更
- `CloudFlare + Firebase Hosting` の構成では、リダイレクトループが発生する可能性がある（理由は後述）。SSL/TLSの暗号化モードをデフォルトのフレキシブルから、フルに変更


このような変更で、Firebase Hosting にデプロイしているサイトへのアクセスにおいて古い暗号スイートの利用をやめさせることに成功しました。

なおCloudFlareの暗号スイート変更に関しては、有料のような記載[^2]もありましたが、ゾーン全体に対しての設定変更はFreeプランで対応可能でした。また今回関係ありませんが、TLS1.3の暗号スイートの変更はできないようです。

[^2]: とおもって記事を書く際に見直したが見つけられなかった

## 実装手順

### 大まかな流れ

大まかな流れは以下のとおりです。

1. Cloudflare のAPIを利用するので、関連情報をあつめる
2. APIを使ってzoneの暗号スイートを変更する
3. zoneの暗号化モードを変更する（リダイレクトループ対策）
4. キャッシュをONにする
5. 通信を確認する

### 1. CloudflareのAPI利用準備

調べた限りですが、ゾーンに適用する暗号スイートを変更する場合はAPI経由で操作を行う必要があります。APIの基本的な操作ですは以下の通りです。

**例）指定されたゾーンのCipher Suites のホワイトリスト設定を取得する**

```bash
# 環境変数の中身は適宜設定してください
CLOUDFLARE_EMAIL=hogehoge@example.com
CLOUDFLARE_API_KEY=*********************
ZONE_ID=***********************

#### Cipher Suites のホワイトリスト設定を取得する
curl https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ciphers \
    -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
    -H "X-Auth-Key: $CLOUDFLARE_API_KEY"

# 何も変更していない場合はvalueの中身が空
# => {"result":{"id":"ciphers","value":[],"modified_on":null,"editable":true},"success":true,"errors":[],"messages":[]}%

```


ここで以下が必要になるのであらかじめ控えておきます。

- CLOUDFLARE_EMAIL
- CLOUDFLARE_API_KEY
- ZONE_ID

`CLOUDFLARE_EMAIL` は、Cloudflareのアカウントに登録している（ログインに使用する）メールアドレスです。

`CLOUDFLARE_API_KEY`は、ダッシュボード（ドメインの設定画面）の右下にある「APIトークンを取得」> Global API Key から取得します。

:::details Global API Keyの確認
![](https://storage.googleapis.com/zenn-user-upload/68359492e61d-20250129.png)
:::

:::message
Global API Keyは非推奨になりつつあり、代わりに**API Tokens**の使用が推奨されています
:::


`ZONE_ID` は、ダッシュボード（ドメインの設定画面）の右下に書いてあります。ドメインごとに変わります。

:::details ZONE_IDの確認

![](https://storage.googleapis.com/zenn-user-upload/849aafe0c9b8-20250129.png)
:::


### 2. CloudFlareのゾーンに対してAPI経由で暗号スイートを変更する

ゾーンに対して暗号スイートを変更します。

DNSの設定でプロキシを有効化している場合（オレンジ色の状態）では、暗号スイートの設定変更がすぐに反映されるので注意が必要です。逆にDNSの設定でプロキシをオフにしていれば通信に影響はありません。

繰り返しになりますが、このAPIはTLS1.2の通信の暗号スイートを変更しています。2025年1月時点ではTLS1.3の暗号スイートは変更できません。

```bash
# 各種環境変数は設定済みとします

### 暗号スイートを変更
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ciphers" \
     -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
     -H "X-Auth-Key: $CLOUDFLARE_API_KEY" \
     -H "Content-Type: application/json" \
     --data '{
       "value": [
         "ECDHE-ECDSA-AES128-GCM-SHA256",
         "ECDHE-ECDSA-AES256-GCM-SHA384",
         "ECDHE-RSA-AES128-GCM-SHA256",
         "ECDHE-RSA-AES256-GCM-SHA384"
       ]
     }'
# => {"result":{"id":"ciphers","value":["ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-RSA-AES256-GCM-SHA384"],"modified_on":null,"editable":true},"success":true,"errors":[],"messages":[]}%


### 暗号スイートを確認
curl https://api.cloudflare.com/client/v4/zones/$ZONE_ID/settings/ciphers \
    -H "X-Auth-Email: $CLOUDFLARE_EMAIL" \
    -H "X-Auth-Key: $CLOUDFLARE_API_KEY"

# => {"result":{"id":"ciphers","value":["ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-RSA-AES256-GCM-SHA384"],"modified_on":null,"editable":true},"success":true,"errors":[],"messages":[]}%
```


どの暗号スイートを使うとよいかは状況にもよりますが、Cloudflareも情報をまとめてくれています。

see: [Recommendations](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/cipher-suites/recommendations/)

### 3. リダイレクトループしないように、SSL/TLS暗号化モードを変更する

デフォルトはフレキシブルになっています。プロキシを有効化し直接オリジンへのアクセス制御をした場合、以下の通信になります。

```
### プロキシON + 暗号化モード：フレキシブルの場合 ###

クライアント
↓
↓ -- HTTPSアクセス
↓
Cloudflare
↓
↓ -- HTTPアクセス（「S」でない点に注意
↓ 
オリジンサーバー
```

オリジンサーバーがFirebase Hostingの場合、HTTPのアクセスを自動的にHTTPSにリダイレクトします。そのため、CloudflareからHTTPアクセスすると、HTTPSにリダイレクトされ続け、リダイレクトループが発生してしまします（経験談）。

回避するために、`SSL/TLS > 概要 > 設定` から、暗号化モードを変更します。Cloudflareとオリジンサーバーの通信をHTTPSにすればよいので、「フル」以上に設定します。

![](https://storage.googleapis.com/zenn-user-upload/5390178c4617-20250129.png)

### 4. DNSの設定を有効化する

`DNS > レコード` からプロキシを有効化します。サブドメインなど用意している場合は、サブドメインなどで動作確認してから本番環境に適用するのが良いと思います。


![](https://storage.googleapis.com/zenn-user-upload/d3862c8dbf4b-20250129.png)

### 5. SSL Labsで暗号スイートが切り替わっていることを確認する

[SSL Labs](https://www.ssllabs.com/ssltest/)を使って暗号スイートが変わっていることを確認します。

CloudFlare経由でアクセスするようにしているため、経路が増えます。

![](https://storage.googleapis.com/zenn-user-upload/fd2fd01d05a4-20250129.png)

TLS 1.2の暗号スイートが切り替わっています。

![](https://storage.googleapis.com/zenn-user-upload/5613f97a1951-20250129.png)

## QA

### 暗号スイートの設定とSSL Labsの結果が異なる。なぜ？

APIで暗号スイートを4つ設定しました。しかしSSL Labsの結果をみると、RSA系の暗号スイートが欠落し2つの暗号スイート許可になっています。

```
# TLS 1.2 (suites in server-preferred order)

・TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
・TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
```

今回Cloudflareのフリープランを利用しています。このためサーバー証明書[^3]はUniversal タイプを利用しています。証明書は `「ECDSA SHA256」` です。

[^3]: Cloudflareの場合エッジ証明書という記載です。

![](https://storage.googleapis.com/zenn-user-upload/c2e6d41069e9-20250130.png)

サーバー証明書に含まれる公開鍵の種類が「ECDSA」のため、TLSハンドシェイク時に「RSA」が使われず、結果的にSSL Labsのスキャン結果から除外されたと理解しました。

### Cloudflare経由のアクセスでAnalyticsのデータ計測は可能か？

こちらは問題なく計測できます。

イベント計測は、オリジンに配置したGoogleのSDKが実行されて計測されるため、キャッシュの影響は受けないという理解です。実際に、キャッシュ導入後もAnalyticsにログが計測されています。

また、アクセス元なども計測できています。これはCloudflareが、`CF-Connecting-IP` ヘッダーでアクセスもとのIPを転送しているようで、それをGoogleのSDKが適切に処理しているからと理解しています。どのようなヘッダーを送りつけているかは公式ドキュメント[^4]を確認ください。

[^4]: [Cloudflare HTTP headers](https://developers.cloudflare.com/fundamentals/reference/http-headers/)

## まとめ

Firebase Hostingを利用しているサービスにおいて、TLS1.2の脆弱な暗号スイートを無効化するため、Cloudflareを活用した対応方法を紹介しました。

以下ポイントです。

- Cloudflareの暗号スイートはAPIで設定変更が必要、また無料プランでも利用可能
- Firebase HostingとCloudflareの組み合わせでは、SSL/TLS暗号化モードを「フル」以上に変更しリダイレクトループを防げる

なお、Firebase Hosting や Cloudflare の設定が変わる可能性はあります。そのため、実際に試す際は SSL Labsなどで実際の暗号スイートの状態を確認することをお勧めします。

## 参考資料
### Cloudflare公式

- [Customize cipher suites](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/cipher-suites/customize-cipher-suites/)
- [CloudFlareのAPI操作方法（zone系）](https://developers.cloudflare.com/api/resources/zones/subresources/settings/)
- [Recommendations](https://developers.cloudflare.com/ssl/edge-certificates/additional-options/cipher-suites/recommendations/)
  - おすすめの暗号スイートのまとめ
- [ERR_TOO_MANY_REDIRECTS](https://developers.cloudflare.com/ssl/troubleshooting/too-many-redirects/)
  - リダイレクトループの話
- [Cloudflare HTTP headers](https://developers.cloudflare.com/fundamentals/reference/http-headers/)
  - Cloudflareがどんなヘッダーをつけて通信を転送しているかのまとめ記事

### その他

- [サードパーティ証明書でのECDSA+RSAのデュアル証明書について](https://community.akamai.com/customers/s/article/JapanUserGroupECDSARSA20220823173303?language=en_US)
  - SSL/TLS証明書（サーバー証明書）の署名方式によってハンドシェイクの暗号スイートが変わるよ、という記載がある
