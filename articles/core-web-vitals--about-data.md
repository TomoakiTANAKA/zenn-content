---
title: "Core Web Vitals対応：フィールドデータとラボデータの違いとデータ計測の方法"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [seo, html, css, ads]
published: true
---

## はじめに

2021年5月[^1]から、Core Web VitalsがGoogleのランキングアルゴリズムに導入される予定です。
有益なコンテンツがランキングのシグナルにおける最も重要な要因である[^2]と言われていますが、
SEOメディアを運営している企業としては、わかりきっていたマイナスポイント（※）は極力潰しておきたいです。

運営しているメディアでは、特に`CLS` のスコアに関して、`PSI` や `Search Console` で警告をうけました。
この記事では、CLSを改善するために

1. 数値をどのように取得し解釈すればよいか
1. 実際にどういった手順で数値を改善していけばよいか

について説明します。（具体的なコードは別の記事で説明します。）

なお、

- Core Web Vitalsとはなんぞや
- LCP / FID / CLSといった指標とは
- パーセントタイルって

といった内容は他に良記事がありますので、こちらでは説明しません。


:::message
実際には減点というよりも、コンテンツや被リンクなどのスコアが同じ場合に優越をつけるといった位置づけと発表されていたと思います。（ソースが見つけられなかった…）
:::

## CLSの数値をどのように取得し解釈すればよいか

### ツール

Core Web Vitalsの数値を見たり、警告や改善方法を受け取る方法はいくつかあります。
かんたんにまとめると、以下のようになります。

全般的にいえることで、数値が出るまでに1〜3分ほど時間がかかります。

| 名称                                                                        |   良い点                | 悪い点            | 
|-----------------------------------------------------------------------------|--------------------------|-----------------|
| [PageSpeedInsight](https://developers.google.com/speed/pagespeed/insights/) | 操作がかんたん（URLいれるだけ）、改善点を教えてくれる | 改善点が多すぎる。結果がでるまで遅い。|
| [SearchConsole](https://search.google.com/search-console/about?hl=ja) | インデックスされているコンテンツ全体で評価してくれる | 1つ1つどう修正すればいいかは教えてくれない |
| [CrUX（Chrome User Experience Report）](https://developers.google.com/web/tools/chrome-user-experience-report?hl=ja)  | レポートになって全体がわかる（フィールドデータのもとデータ） | リアルタイム性はない |
| LightHouse                                                                  | Core Web Vitals以外の評価もしてくる | 結果がでるまで遅い |
| Chrome Developer Tools                                                      |  CLSの発生ポイントなどHTMLのレンダリングに即して表示してくれる。ネットワークシュミレーションができる。  |　結果がでるまで遅い。操作が難しい。エンジニア以外では厳しい気がする |
| その他（ChromeExtension等）                                                    |   導入がかんたんで、結果もわかりやすい | 結果が正しいとは限らない（別プラグイン等と干渉して正しい結果がでないことがある） |

上記ツールの中でPSIが最も利用しやすく、情報も豊富なこともあり、私は基本的にPSIを利用してCLSの改善を行いました。

### PSIでCLSを計測する

![PSIサンプル](https://storage.googleapis.com/zenn-user-upload/thybr6f2qle9cr59uc5mvgot9jue)

検索窓にURLを入れるだけでいいので、利用方法は簡単です。
一方、「フィールドデータ」「オリジンの概要」「ラボデータ」という聞き慣れない用語がでてきます。

きちんと用語を理解しないと、数値上は改善したと思っても、Search Console上での警告は終わらないといったことになりかねません[^3:僕のことです]。
きちんと理解して、改善をしていきましょう。

やはり基本は公式ドキュメントが良いです。
[PageSpeed Insights API について](https://developers.google.com/speed/docs/insights/v5/about)

#### 「フィールドデータ」
実際にユーザーがアクセスした環境で取得した数値です。
ネットワークスピードや、読み込み中のスクロールなど考慮された数値です。

実際のユーザーアクセスを元に計測した値なので、アクセスが少ないと数値が計上されません。

後述するラボデータと数値の傾向は似ますが、完全には一致しません。（ラボデータがシミュレーション値のため）

#### 「オリジンの概要」

自分自身のサイトの全体的な傾向です。
WordPress等のCMSでコンテンツを生成している場合、どの記事の同様な形式になるので、フィールドデータと傾向は一致するはずです。

#### 「ラボデータ」

PSIを実行した環境におけるシミュレーション環境での数値です。
読み込み中のスクロールなど考慮されないようです。
僕の経験ですが、特にCLSに関していえば、スクロールなしのfirstViewを見てCLSを計算しているようでした。

ラボデータのCLSの値が0になったからといって、描画時にCLSが発生していないわけではないので注意が必要です。

## CLSの数値をどのように計測すればよいか

基本的には、
- 修正
- PSIでラボデータを確認する
の繰り返しです。

ただ、PSIはbasic認証配下でアクセスできなかったり、CLSがシミュレーション値でfirstViewしか見ていない、ので注意が必要です。
また、開発環境でも数値の改善を確認することができません。

ネットワークスピードを遅くして、目視確認でも十分かなと思いますが、API[^3]を利用することも可能です。

APIは`<body>`タグよりも前にCLS APIを読み込んでおけば、自動的にconsoleにログを吐き出してくれます。
ページ読み込み時に、カーソル移動するなどすると、実際表示していた位置でのCLSの値を表示してくれます。

```html
<script>
addEventListener("load", () => {
    let DCLS = 0;
    new PerformanceObserver((list) => {
        list.getEntries().forEach((entry) => {
            if (entry.hadRecentInput)
                return;  // Ignore shifts after recent input.
            DCLS += entry.value;
        });
    }).observe({type: "layout-shift", buffered: true});
});
</script>
```

## 最後に

僕のようにラボデータだけみて改善しきったと勘違いしないように注意してください。

CLSの発生原因に関しては、「画像のアスペクト比の指定がない」・「広告」が主な原因だと思います。
こちらは主にCSSで対応することになりますが、詳しい方法は別の記事で説明します。

[^1]: [Google 検索へのページ エクスペリエンスの導入時期](https://developers.google.com/search/blog/2020/11/timing-for-page-experience)
[^2]: [コンテンツを最適化する](https://developers.google.com/search/docs/beginner/seo-starter-guide#make-your-site-interesting-and-useful)
[^3]: [layout-instability](https://github.com/WICG/layout-instability)