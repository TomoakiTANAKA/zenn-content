---
title: "Core Web Vitals対応：CLS主な原因である画像と広告の修正方法"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [seo, html, css, ads]
published: true
---

## はじめに

CLSの発生原因に関しては、「画像のアスペクト比の指定がない」・「広告」が主な原因だと思います。（経験談）。
こちらは主にCSSで対応することになります。

## CLS発生の基本的なメカニズムと対策

CLSの発生メカニズムは下記の通りです。

1. HTML読み込み時には高さが確定しない要素がある（画像や広告。例えば画像のURLからは、実際の高さや幅はわからない。）
1. この時点では要素の高さは0扱いされている。
1. HTMLレンダリング後に、実際の要素（画像や広告）のロードが発生、高さが決まる
1. もともと0の高さだったところに要素が「挿入」され、コンテンツ全体がシフトしたように見える

対策は、予め「4.」で取得する高さを予測して、cssで高さを指定することです。

:::message warn
この記事ではスタイルははsaccで記述しています。（作者の好み）
:::


## CLS対応〜画像編〜

画像は、端末によって表示サイズが変えることが多いと思います（レスポンシブ対応）。

- HTMLで画像のアスペクト比[^1]の指定
- cssでレスポンシブ対応

という方法で高さを予約します。


```html
<!-- 3:2 の比率を指定しておく -->
<img class="article-header__image" alt="魚　タイ　鯛　さばき方　下処理　うろこ　ウロコ取り" width="480" height="320" 
src="https://d17uhz2kob7es4.cloudfront.net/images/pictures/images/000/022/196/shutterstock_260564267-thumb_480.jpg?1617095560">
```

```scss
.article-header {
  &__image {
    width: 100%;
    height: auto;
    max-height: 425px;
    object-fit: contain; // 縦長画像でもサイズを比率を維持して枠にいれる
  }
}
```

## CLS対応〜広告（アドセンス）編〜

アドセンスが最も一般的だと思うので、そちらで説明します。
cssの指定がない場合、初期の高さが0になってしまうので、広告枠の周りに予め「最小でこれくらになるよという高さ」を予約しておきます。

```html
<div class="responsibe-adsense-wrap">
  <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
  <ins class="adsbygoogle"
       style="display:block"
       data-ad-client="ca-pub-xxxxxxxxxxxxx"
       data-ad-slot="xxxxxxxxxxxxxx"
       data-ad-format="rectangle"
       data-full-width-responsive="true"></ins>
  <script>
    (adsbygoogle = window.adsbygoogle || []).push({});
  </script>
</div>
```

```scss
// 広告サイズの想定に関しては
// できる限り大きいもの > 336×280 > 300x250 の優先度を想定
.responsibe-adsense-wrap {
  width: auto;
  min-height: 280px;
  height: auto;

  text-align: center;

  @media screen and ( max-width: 339px ) {
    width: 300px;
    min-height: 250px;
    height: auto;
  }
}
```

CLSのフィールドデータを改善する、という観点では

- ①ページの表示速度を上げ、スクロールする間もなくコンテンツを表示させきってしまう
- ②firstViewに広告をいれない

という対応をすることでも数値は改善すると思います。

実際にgoogleのドキュメントでもfirstViewで広告は使わないほうがいいよ、とサジェストされています。


## 最後に

今回は基本的なCLS対策を紹介しました。発生の原理はシンプルなので、比較的理解しやすいと思います。
収益的な観点ですと、GoogleAdManager（GAM）やHeaderBidding等の広告テクノロジーを使う場合も多いと思います。

私の取り扱っているサービスでもGAMを使った枠を配信しているのですが、
こちらは、予め広告枠のサイズをJavaScriptで指定しているにも関わらず、レポートを見るとそれ以外のサイズの広告も多く配信されていました。

弊社の場合、このような場合は、レポートから表示頻度の高いサイズを救えるように、最小サイズを決定しました。
このあたりは定期的にCore Web VitalsやGAMのレポートを見て、調整していくしかなち思っています。

いい方法をご存知の方がいらしたらぜひ教えてほしいです。

## 参考文献
- [Optimize Cumulative Layout Shift](https://web.dev/optimize-cls/)

[^1]: [Modern best practice](https://web.dev/optimize-cls/#modern-best-practice)
