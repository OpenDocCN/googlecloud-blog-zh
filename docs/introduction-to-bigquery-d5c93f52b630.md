# BigQuery 简介

> 原文：<https://medium.com/google-cloud/introduction-to-bigquery-d5c93f52b630?source=collection_archive---------0----------------------->

*最初发表于*【https://pbhadani.com】

*在这篇文章中，我将谈论谷歌的大数据分析 [BigQuery](https://cloud.google.com/bigquery/) 服务。*

*![](img/ec3b6fb89dc1ef31e35ea482f60d121c.png)*

*图片来自 [Pixabay](https://pixabay.com/) 的[像素](https://pixabay.com/users/Pexels-2286921)*

# *概观*

*[BigQuery](https://cloud.google.com/bigquery/) 是一款基于谷歌云平台的无服务器、高可伸缩、高性价比的企业级现代数据仓库产品。它允许分析师使用 ANSI SQL 快速分析数 Pb 的数据，而没有运营开销。*

# *关键特征*

***无服务器:**无运营模式。Google 在幕后管理所有的资源供应。*

***快速 SQL:** 支持亚秒级查询响应时间和高并发的 ANSI SQL。*

***托管存储:**数据一旦加载到 BigQuery，就以有效的方式存储在由 BigQuery 管理的&中。*

***数据加密&安全性:**数据静态加密，集成云端 IAM，安全性高。*

***BigQuery ML:** 支持数据科学家和数据分析师使用 SQL 语法在 BigQuery 中构建、训练和测试 ML 模型。*

***BigQuery GIS:** 通过允许分析和可视化 BigQuery 中的地理空间数据来实现位置智能。*

***灵活定价模式:**按需和统一费率定价。最新定价模式，请参考[官方文档](https://cloud.google.com/bigquery/pricing)*

*最新完整列表，请参考[官方文档](https://cloud.google.com/bigquery#section-9)*

# *如何访问 BigQuery？*

*有多种方式可以与 BigQuery 交互:*

*   *[网络用户界面](https://console.cloud.google.com/bigquery)*
*   *[BQ 命令行工具](https://cloud.google.com/bigquery/docs/bq-command-line-tool)*
*   *[BigQuery API 客户端库](https://cloud.google.com/bigquery/docs/reference/libraries)*
*   *[BigQuery REST API](https://cloud.google.com/bigquery/docs/reference/rest)*
*   *[第三方工具](https://cloud.google.com/bigquery#section-11)*

# *使用 bq 与 BigQuery 交互*

## *先决条件*

*本帖假设如下:
1。我们已经有了一个 GCP 项目和启用的 big query API。
2。谷歌云 SDK ( `gcloud`)。如果没有，那就参考我之前的博客-[Google Cloud SDK 入门](https://pbhadani.com/posts/getting-started-wth-google-cloud-sdk/)。*

# *BigQuery 操作*

## *创建数据集*

```
*bq mk bq_dataset*
```

## *列出数据集*

```
*bq ls*
```

*输出:*

```
*datasetId
--------------
bq_dataset*
```

## *在数据集中创建表*

```
*bq mk **\** --table **\** --expiration **3600** **\** --description **"This is my BQ table"** **\** --label env:dev **\** bq_dataset.first_table **\** col1:STRING,col2:FLOAT,col3:STRING*
```

## *检查 BigQuery 表*

***注意:**我将检查公共数据集中的一个表。*

```
*bq show bigquery-public-data:covid19_jhu_csse.summary*
```

*输出:*

```
*Table bigquery-public-data:covid19_jhu_csse.summary

Last modified              Schema              Total Rows   Total Bytes   Expiration   Time Partitioning   Clustered Fields      Labels
----------------- ----------------------------- ------------ ------------- ------------ ------------------- ------------------ --------------
07 Jun 10:06:41   |- province_state: string     254940       41005062                                                          freebqcovid:
                |- country_region: string
                |- date: date
                |- latitude: float
                |- longitude: float
                |- location_geom: geography
                |- confirmed: integer
                |- deaths: integer
                |- recovered: integer
                |- active: integer
                |- fips: string
                |- admin2: string
                |- combined_key: string*
```

## *运行查询*

```
*bq query --use_legacy_sql=**false** **\
'SELECT
date,
country_region,
SUM(confirmed),
SUM(deaths)
FROM
`bigquery-public-data.covid19_jhu_csse.summary`
GROUP BY
date,
country_region
HAVING date = "2020-05-31"
AND
country_region IN ("India", "US")'***
```

*输出:*

```
*Waiting on bqjob_r2935edb2e19bc9f_000001728e32aded_1 ... (0s) Current status: DONE
+------------+----------------+---------+--------+
|    date    | country_region |   f0_   |  f1_   |
+------------+----------------+---------+--------+
| 2020-05-31 | US             | 1790172 | 104381 |
| 2020-05-31 | India          |  190609 |   5408 |
+------------+----------------+---------+--------+*
```

## *清理:删除数据集*

```
*bq rm -r bq_dataset*
```

*希望这篇博客能帮助你熟悉 BigQuery。*

*如果您有反馈或问题，请通过 [LinkedIn](https://linkedin.com/in/pradeepbhadani) 或 [Twitter](https://twitter.com/bhadanipradeep) 联系我。*