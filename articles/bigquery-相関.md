---
title: "BiqQueryを利用してセッション数を求める"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sql, firebase, bigquery]
published: true
---

## 継続率を求める 
### 最終形のテーブル

どんな感じのデータなの？
日にち	翌日継続率	3日後継続率	7日後継続率	14日後継続率
2013/01/01	28.4%	23.1%	20.2%	19.4%
2013/01/02	24.1%	21.4%	16.3%	15.3%
2013/01/03	27.7%	25.5%	19.5%	15.4%
2013/01/04	30.2%	23.3%	21.6%	16.5%
2013/01/05	25.4%	22.5%	17.4%	14.9%

### 集計用テーブル

*sessionsテーブル*

| user_id       | access_date    | access_date_unix    |
|----------|--------------------------|
| xxx | xxx |xxx |
| xxx | xxx |xxx |
| xxx | xxx |xxx |
| xxx | xxx |xxx |
| xxx | xxx |xxx |
| xxx | xxx |xxx |

### 注意点

- 自前でログインIDを付与するシステム・サービスではないので、user_idとして `user_pseudo_id` を利用しています

## SQL
### 準備

### cohort

```sql
-- 検索期間の指定、DATE_SUB / CURRENT_DATE を利用して動的に計算する
CREATE TEMPORARY FUNCTION fromDate() AS (
  -- FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), interval 15 DAY))
  FORMAT_DATE('%Y%m%d', '2022-09-01')
);

CREATE TEMPORARY FUNCTION toDate() AS (
  FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), interval 1 DAY))
);

-- visitStartTimeを日本時間の日付に変更する関数
CREATE TEMPORARY FUNCTION TO_DATE(a_timestap INT64) AS (
  DATE(TIMESTAMP_MICROS(a_timestap), 'Asia/Tokyo')
);

CREATE TEMPORARY FUNCTION TO_UNIX_DATE(a_date DATE) AS (
  UNIX_DATE(a_date)
);


WITH repeat_interval AS (
  SELECT
      CONCAT(id, " day repeat") AS index_name
    , CAST(id AS INT64) AS interval_date
  FROM
    unnest(["01", "02", "03", "07", "14", "28"]) AS id
)

-- [集計用マスタ] installs テーブル
-- アプリの初回利用日を登録日とみなす
, installs AS (
  SELECT
      user_pseudo_id AS user_id
    , DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo') AS installed_date
    , UNIX_DATE(DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo')) AS installed_date_unix
  FROM
    `akamaru-49aa3.analytics_309857157.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN fromDate() AND toDate()
    AND event_name = 'first_open'
)

-- [集計用マスタ] action_log テーブル
-- @memo
-- このSQLでは、 session_start のイベントを起点にしている
-- サービスによって、起動（ちゃんと触った）の定義は異なるはずなので、適宜修正する
, action_log AS (
  SELECT
      user_pseudo_id AS user_id
    , TO_DATE(event_timestamp) AS launched_date
  FROM
    `akamaru-49aa3.analytics_309857157.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN fromDate() AND toDate()
    AND event_name = 'session_start'
  GROUP BY
      launched_date
    , user_pseudo_id
)

-- [集計用] ユーザーがいつインストールして、いつアプリを触ったかを管理するテーブル
-- install + action + repeat_interval を クロスジョインして、リピート間隔ごとの日付データを作る
, action_log_with_hoge AS (
  SELECT
      i.user_id
    , i.installed_date
    , a.launched_date
    , MAX(a.launched_date) OVER() AS latest_date

    -- クロス集計で、全データ x 継続率を計測したい期間分の日付データを作る
    , r.index_name
    , DATE_ADD(i.installed_date, INTERVAL r.interval_date DAY) AS index_date
  FROM
    installs AS i
  LEFT OUTER JOIN
    action_log AS a
  ON
    i.user_id = a.user_id
  CROSS JOIN
    repeat_interval AS r
)

-- [集計用] 日付
, user_action_flag AS (
  SELECT
      user_id
    , installed_date
    , index_name
    , SIGN(
      SUM(
        CASE WHEN index_date <= latest_date THEN
          CASE WHEN index_date = launched_date THEN 1 ELSE 0 END
        END
      )
    ) AS index_date_action
  FROM
    action_log_with_hoge
  GROUP BY
      user_id
    , installed_date
    , index_name
    , index_date
)

-- SQL本体
SELECT
    installed_date
  , index_name
  , AVG(100.0 * index_date_action) AS repeat_rate
FROM
  user_action_flag
GROUP BY
  installed_date, index_name
ORDER BY
  installed_date, index_name
;
```


モチベーション
・BigQueryは、データスキャンする量が増えれば増えるほどお金がかかる認識
・またSQLが長くなるとメンテナンスが大変

例えば、「8月のインストールユーザー」だけ
9月のインストールユーザー