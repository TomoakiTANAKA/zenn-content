---
title: "BiqQueryでよく使う処理を関数化する"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sql, firebase, bigquery]
published: true
---

よく使う関数（特に日付系）をまとめる
まとめて使いまわしやすくなる
本体のSQLのコアの稼働性をあげる

## 形式


方はここで確認
https://support.google.com/firebase/answer/7029846?hl=ja


## 日付関連
### 横断的に
```sql
-- 検索期間の指定、DATE_SUB / CURRENT_DATE を利用して動的に計算する
CREATE TEMPORARY FUNCTION fromDate() AS (
  FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), interval 15 DAY))
);
```

```sql
CREATE TEMPORARY FUNCTION fromDate() AS (
  FORMAT_DATE('%Y%m%d', DATE('2022-09-01'))
);
```

## OS関連

SELECT
  date(timestamp_micros( event_timestamp ),"Asia/Tokyo") as date
  , count(distinct user_pseudo_id) AS COUNT
FROM `akamaru-xxxxx.analytics_xxxxxxxxx.events_*`
WHERE event_name = 'first_open'
AND _TABLE_SUFFIX BETWEEN "20220428" AND "20220430"
group by date;
```

timestamp_micros
user_pseudo_id
first_oepn
_TABLE_SUFFIX


## DAU / WAU / MAU

指定された期間を切り取ることで表現します

```sql
SELECT
  count(distinct user_pseudo_id) AS COUNT
FROM `akamaru-49aa3.analytics_309857157.events_*`
WHERE event_name = 'user_engagement'
-- AND platform = "ANDROID"
-- AND platform = "IOS"
AND (platform = "IOS" OR platform = "ANDROID")
AND _TABLE_SUFFIX BETWEEN "20220501" AND "20220531"
-- AND _TABLE_SUFFIX BETWEEN FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 28 DAY)) AND FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
```

## 参考

- [BigQueryでユーザー定義関数（UDF）は武器になるという話](https://techblog.zozo.com/entry/bigquery-udf)
  - ZOZOさんのtechブログ