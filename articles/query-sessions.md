---
title: "BiqQueryを利用してセッション数を求める"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sql, firebase, bigquery]
published: true
---

## 概要

- BigQuery初心者向けに、Session数の求め方を、できるだけ再現可能なように説明する

## 想定読者

- 基本的なSQLの記述ができる
- BigQueryについてなんとなく知識がある
- 普段GA4やFirebaseのダッシュボードを利用している


## 注意

- テーブル名 `your_datasets.analytics_123456789.events_*` 部分は適宜読み替えてください。

## 基本
### firebaseにおけるセッション数

セッション開始時に `session_start` イベントが落ちます。

同時に
- セッション ID（ga_session_id）
- セッション番号（ga_session_number）
が生成されます。

GoogleAalytics4やFirebaseのダッシュボードは、 `ga_session_id` をベースにセッション数を算出しています。

## ダッシュボードとBigQueryで算出した数値には乖離がある

ドキュメントにも記載がありますが、微妙に数値に乖離がでます。
月並みですが、分析や意思決定において混乱しないように、事前にどちらを利用するか決めておくのが良いと思います。

僕の環境では、BigQueryの値が少しだけ小さくなりました（-1.5%程度）。


## BigQueryを利用してセッション数を求める
### 考え方

サブクエリを使って、集計しやすいテーブルを作成してから、集計を行います。


| access_date       | sid                    |
|----------|--------------------------|
| 2022-09-01 | hogehoge |
| 2022-09-01 | fugafuga |

その他注意したこと

- `ga_session_id` ベースに集計
- セッション数は日付（YYYY-MM-DD）レベルの集計で十分、`DATE` 関数を利用して、日付フォーマットに変化する
- [ドキュメント](https://support.google.com/firebase/answer/9191807?hl=ja)を参考に、`ga_session_id` ごとに集計
- 他の集計などにも利用しやすいようにいくつか処理をまとめる

### SQL

```sql
-- 計算範囲指定、BigQueryのテーブル形式に合わせて「YYYYMMDD形式」の文字列をつくる
-- WHERE部分で、「20220901」と直接指定しても良いが、いくつも日付範囲を指定する場合は、関数として定義すると便利
CREATE TEMPORARY FUNCTION fromDate() AS FORMAT_DATE('%Y%m%d', DATE('2022-09-01'));
CREATE TEMPORARY FUNCTION toDate() AS FORMAT_DATE('%Y%m%d', DATE('2022-09-01'));

-- sessionテーブルの作成
WITH sessions AS (
  SELECT
    DATE(TIMESTAMP_MICROS(event_timestamp), 'Asia/Tokyo') AS access_date,
    (
      SELECT
        value.int_value
      FROM
        UNNEST(event_params)
      WHERE
        key = 'ga_session_id'
    ) AS sid
  FROM
    -- BigQueryのデータは日付ごとに分かれているため、複数日にまたがる検索をする場合は、ワイルドカードを使う
    `your_datasets.analytics_123456789.events_*`
  WHERE
    _TABLE_SUFFIX BETWEEN fromDate() AND toDate()
    AND event_name = 'session_start'
)

-- session数の集計
SELECT
  access_date
  , COUNT(DISTINCT sid) AS session_count
FROM sessions
GROUP BY access_date
ORDER BY access_date
```

## 参考

- [[GA4] アナリティクス セッションについて](https://support.google.com/firebase/answer/9191807?hl=ja)
- [タイムスタンプ関数](https://cloud.google.com/bigquery/docs/reference/standard-sql/timestamp_functions?hl=ja)
- [TIMESTAMP_MICROS](https://cloud.google.com/bigquery/docs/reference/standard-sql/timestamp_functions?hl=ja#timestamp_micros)
