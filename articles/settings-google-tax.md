---
title: "【PlayStore】アメリカ合衆国の税務情報の入力ガイド"
emoji: "🔰"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [settings, google, playstore, android]
published: true
---

![](https://storage.googleapis.com/zenn-user-upload/d83442b5dd1a-20230131.png)

## 税務情報とは？なぜ申請する必要があるの？

私達が開発したアプリや、投稿した動画などは、Googleのサービスを通して世界中に配信されています。
アプリや動画などで収益を上げている場合、日本だけでなく、アメリカやその他の国々でも収益が発生する可能性があります。

事業者（ここでは法人や個人）は、収益に対して税金を支払う必要があります。
一方、収益を支払う企業（ここではGoogle）は、事業者がきちんと税金を仕払はないと、政府から罰則をうけることがあります。

そうならないように事前に収益を支払う企業が、収益から収めるべき税金を引いておくのが源泉徴収です。

しかしそうなると、

- Googleからの収益に対して税金をアメリカ政府に支払う（源泉徴収として）
- 源泉徴収を除く収益を売上金として、もろもろ差し引いて、日本政府に税金を支払う

と2つの国に税金を収めることになります。

Googleに税務情報を提出すると、 アメリカ政府への源泉徴収支払いを回避することができます。

## 事前に準備しておくもの

- TIN：Tax Identification Number
  - 納税者と特定するための番号のこと 
  - アメリカ合衆国の源泉徴収を0%にする場合に必要

:::message
日系企業の場合は法人番号、個人事業主や個人の場合はマイナンバーで代用可能です
:::

- 企業名や事業体がわかるような資料（いずれか、pdf）
  - 企業設立登記書
  - 登録証明書
  - 納税者証明書
  - 定款
  - 会社定款

:::message
フォームの特性上、日本語を入力できません。
ですが、事前に登録している企業情報（アカウント情報）と差異があると怒られます。
最後にuploadする必要があるので、事前に用意しておきましょう。
:::

## 前提条件および注意事項

- 法人企業として税務情報を入力する場合のガイドです
- 個人事業主向けの説明もありますが、適宜読み替えてください
- どのような意図で項目を選択しているか記載しますが、最終的な選択は自己責任でお願いします

## アメリカ合衆国の政務情報の入力方法

基本的には、ガイドに従って入力していくだけです。
ただ、聞き慣れない文言もありますので、適宜説明を入れています。

![](https://storage.googleapis.com/zenn-user-upload/9a2e0c0ba183-20230131.png)

ポップアップの指示に従って、[税務情報の追加] から 開始します。
GooglePlayConsoleにログイン、サイドバーの[お支払いプロファイル]からも設定できます。

### ①米国の税務情報

![](https://storage.googleapis.com/zenn-user-upload/1f133f5740ac-20230131.png)

#### 【1】講座の種類はなんですか？
> このアカウントは、DBA（ビジネス形態）名またはみなし事業体ですか？

法人の場合、`非個人/事業体`をチェックします。（個人事業主の場合は、`個人` をチェックします。）

DBAは、屋号を使っているかの質問です。 個人事業主の場合はチェックする場合があると思います。

今回は法人での契約なので、`チェックなし`で進みます。

:::message
DBA：Doing Business As の略称
:::

#### 【2】その事業体は米国の組織または法人ですか？

米国の組織・法人ではないので、`いいえ`を選択します

#### 【3】W-8 納税申告用紙タイプを選択

説明をよく読み適切なものを選択します。
日系企業で主な事業先が国内（日本）の場合は、`W-8BEN-E` が適切かと思います。

### ②W-8BEN-E 納税フォーム

:::message alert
選択肢は日本語なのですが、 入力欄には日本語が使えません。
:::

#### 【1】納税者番号

![](https://storage.googleapis.com/zenn-user-upload/c1b8d1820f5d-20230131.png)

- `組織名` は英語で入力
  - 後述の`事業体の種類`を選択するので、株式会社等の表記は不要（そもそも入力できない）
- `DBA` は法人の場合ないはずなので、空白
- `法人または組織の国/地域`は、日系企業なので`日本`を選択
- `事業体の種類` は`企業`を選択（個人の環境に合わせてください）

**納税者番号**
納税者番号（TIN）を入力しておかないと、後述する源泉徴収の特例を受けとることができません。必ず入力してください。
企業の納税番号は、[国税庁](https://www.houjin-bangou.nta.go.jp/)のページから調べることができます。

#### 【2】住所

![](https://storage.googleapis.com/zenn-user-upload/ad3fc81d2620-20230131.png)

ガイドに従って入力します。
住所は英語表記で入力します。日本語表記するとエラーになります。

#### 【3】租税条約

> 租税条約下で源泉徴収に適用される軽減税率の請求を行っていますか？

1. `はい` を選択
2. `米国との租税条約の適用のある国 / 地域の居住者` から `日本` を選択
3. `条約上の優遇措置が請求されている項目の所得があり、該当する場合は、優遇措置の制限に関する条約上の要件を満たしている` を選択
4. `所有 / 税源侵食テストを満たす会社` を選択（※ 環境によって読み替えてください）

![](https://storage.googleapis.com/zenn-user-upload/623faaa02373-20230131.png)

> 特別な料率や条件

今後受け取る可能性のあるものをまとめて申請できるので、まとめて申請しておきましょう。
前述したとおりですが、`TIN`を入力しておかないと、次へはすすめません。

![](https://storage.googleapis.com/zenn-user-upload/ec1c58b9b97f-20230131.png)

#### 【4】書類のプレビュー

内容を確認して次に進みます。

![](https://storage.googleapis.com/zenn-user-upload/cbfb1349f524-20230131.png)

#### 【5】納税証明

内容を確認し、署名します。次に進みます。

![](https://storage.googleapis.com/zenn-user-upload/7bfc112ea9c0-20230131.png)

#### 【6】米国内で行っている活動とサービス、および宣誓供述書

![](https://storage.googleapis.com/zenn-user-upload/5f4c4bcd0c52-20230131.png)

> 納税者番号 セクションに記されている事業体は、これまでに米国内で Google を対象にした活動やサービスを行ったことがありますか？

```
米国居住者が米国内からウェブサイトにアクセスしたり、
動画を視聴したりすることは、通常、米国における活動とサービスには含まれません。
```

とあるので、米国に住んでいる方が自社のプロダクトやWebサービスにアクセスする場合は条件に含まれません。

`いいえ`を選択します。

> 税務情報を提出するのは、お支払いを受け取ったことがない新規または既存のお支払いプロファイルですか？それとも過去にお支払いを受け取ったことがある既存のお支払いプロファイルですか？

#### 【7】税金に関するレポート

ペーパーレスで十分かと思うので、`ペーパーレス`を選択しておきます。

![](https://storage.googleapis.com/zenn-user-upload/87478eb5473a-20230131.png)

### ③確認書類の提出

![](https://storage.googleapis.com/zenn-user-upload/6e48662699c7-20230131.png)

会社名と登録名が異なる場合などに、追加で書類を提出する必要があるようです。 
日本語である「株式会社」などをフォームに入力できないので、絶対に一致しない気もしますが…

念のため登録しておきます。

## 完了報告とステータス

[お支払いプロファイル]で審査中の状態になります。

![](https://storage.googleapis.com/zenn-user-upload/c6b2545515a6-20230131.png)

## まとめ

売上から米国の源泉徴収で30%自動で引かれないようにするために、きちんと申請をしておきましょう。

## 参考文献
- [U.S. Taxpayer Identification Number Requirement](https://www.irs.gov/individuals/international-taxpayers/us-taxpayer-identification-number-requirement)
  - TINの要件をまとめた公式資料（英語）
- [Information on Tax Identification Numbers(Japan)](https://www.oecd.org/tax/automatic-exchange/crs-implementation-and-assistance/tax-identification-numbers/Japan-TIN.pdf)
  - 日本企業や個人で、TINになる要件を記載しているPDF、マイナンバーや法人番号がTINになるよ、ということがかいてある
- [国税庁法人番号公表サイト](https://www.houjin-bangou.nta.go.jp/)
  - 法人番号を調べるためのサイト
