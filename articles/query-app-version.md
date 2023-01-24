---
title: "[BiqQuery] iOSやAndroidでリリースした特定バージョンに絞ってイベントを集計する方法"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sql, firebase, bigquery]
published: true
---

## まとめ

- iOSやAndroidのアプリバージョンは、`app_info.version` に、`2.5.0`のような形式で`文字列`として格納されている
- iOSやAndroidといったプラットフォームは、`platform` に文字列で`IOS`や`ANDROID`という形式で`文字列`として格納されている

## 想定読者

- 基本的なSQLの記述ができる
- BigQueryについてなんとなく知識がある
- 普段GA4やFirebaseのダッシュボードを利用している

## 注意

- テーブル名 は適宜読み替えてください
  - `your_datasets.analytics_123456789.events_*` 部分

## BigQueryで特定のバージョンにおけるイベント数をもとめる
### クエリの概要

- 複数のイベントテーブルを跨いで検索する。この時日付処理を使いまわしたいので`CREATE TEMPORARY FUNCTION`で関数を定義
  - `DATE_SUB` を利用して、常に最新のN日間の情報を取得するといった使い方も良い
  - 特定の日付で絞り込みたい場合は、コメントアウトした部分を参考にして絞り込む
- `WITH` を利用して、version毎のテーブルを個別に作成しておく
  - `event_name` の絞り込みは、筆者環境でのもの。各自の要件に合わせて絞り込み条件変更のこと。 

### クエリ本体

```sql
-- イベント絞り込み開始日付
-- BigQueryのテーブル形式に合わせて「YYYYMMDD形式」の文字列をつくる
-- WHERE部分で、「20230101」と直接指定しても良いが、関数として定義すると使いまわせたり、SQL本体がスッキリするので便利
CREATE TEMPORARY FUNCTION fromDate() AS (
  FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), interval 15 DAY))
  -- FORMAT_DATE('%Y%m%d', '2023-01-01')
);

-- イベント絞り込み終了日付
CREATE TEMPORARY FUNCTION toDate() AS (
  FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), interval 1 DAY))
  -- FORMAT_DATE('%Y%m%d', '2023-01-31')
);

-- firebaseのtimestampを日本時刻に変換するヘルパー
CREATE TEMPORARY FUNCTION TO_DATE(a_timestap INT64) AS (
  DATE(TIMESTAMP_MICROS(a_timestap), 'Asia/Tokyo')
);

-- [集計用マスタ] events テーブル
WITH events AS (
  SELECT
    TO_DATE(event_timestamp) AS dt
  , event_name AS event_name
  , COUNT(event_name) AS count
  FROM
    -- BigQueryのデータは日付ごとに分かれているため、
    -- 複数日にまたがる検索をする場合は、ワイルドカードを使う
    -- （※）テーブル名は定義読み替え
    `your_datasets.analytics_123456789.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN fromDate() AND toDate()
  -- （※）イベント絞り込み条件は適宜読み替え
  AND event_name LIKE "show_screen_game%"
  -- platformでOS / app_info.version でアプリのバージョンを絞り込める
  AND platform = 'IOS'
  AND app_info.version = '2.5.0'
  GROUP BY
    event_name, dt
  ORDER BY
    dt
)

-- 本体のSQL
SELECT
  *
FROM
  events
;
```

## 参考文献

- [[GA4] BigQuery Export スキーマ](https://support.google.com/firebase/answer/7029846?hl=ja)
  - `app_info` の要素を確認することができます
