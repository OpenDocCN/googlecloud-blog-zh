# BigQuery:从增量到最新-所有历史

> 原文：<https://medium.com/google-cloud/bigquery-delta-to-latest-all-history-c8f28f889c91?source=collection_archive---------1----------------------->

# 挑战

构建在 [BigQuery: delta to latest](/google-cloud/bigquery-deltas-to-latest-e84999fdb311) 之上，而不是进行多次更改并找到我们想要的最新值**来捕捉随时间的更改。**

不使用 [ARRAY_AGG](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#array_agg) ，而是使用 [LAST_VALUE](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#last_value) 作为分析函数。

# 解决方案

## 测试数据

首先，我们将创建一组名为 testing.sparsedata 的测试数据。这将有一个唯一的键 **id** ，一个时间戳 **ts** ，以及包含随机数据或 NULL 的列 **c1** 到 **c9** 。

```
CREATE OR REPLACE TABLE `testing.sparsedata` AS
SELECT
  CAST(FLOOR(RAND()*1000) AS INT64) AS id,
  TIMESTAMP_ADD(
    CURRENT_TIMESTAMP(),
    INTERVAL CAST(FLOOR(RAND()*1000) AS INT64) SECOND) AS ts,
  IF(RAND()>0.75, CAST(RAND() AS STRING), NULL) AS c1,
  IF(RAND()>0.75, CAST(RAND() AS STRING), NULL) AS c2,
  IF(RAND()>0.75, CAST(RAND() AS STRING), NULL) AS c3,
  IF(RAND()>0.75, CAST(RAND() AS STRING), NULL) AS c4,
  IF(RAND()>0.75, CAST(RAND() AS STRING), NULL) AS c5,
  IF(RAND()>0.75,
    CONCAT("id=", CAST(RAND() AS STRING)),
    NULL) AS c6,
  IF(RAND()>0.75,
    CONCAT("id=", CAST(RAND() AS STRING)),
    NULL) AS c7,
  IF(RAND()>0.75,
    CONCAT("id=", CAST(RAND() AS STRING)),
    NULL) AS c8,
  IF(RAND()>0.75,
    CONCAT("id=", CAST(RAND() AS STRING)),
    NULL) AS c9
FROM
  UNNEST(GENERATE_ARRAY(1, 1000000)) AS d
```

## 询问

将该数据转换为一系列记录，其中对于每一列 **c1** 到 **c9** ，它具有**最近的非空数据**可以用 LAST_VALUE 和 IGNORE NULLs 分析函数来完成。

在这种情况下，命名窗口也用于简化 SQL 的编写，但也可能是一种优化，即窗口可以只计算一次。

查询如下所示:

```
SELECT
  id,
  ts,
  LAST_VALUE(c1 IGNORE NULLS) OVER (sparse_data) AS c1,
  LAST_VALUE(c2 IGNORE NULLS) OVER (sparse_data) AS c2,
  LAST_VALUE(c3 IGNORE NULLS) OVER (sparse_data) AS c3,
  LAST_VALUE(c4 IGNORE NULLS) OVER (sparse_data) AS c4,
  LAST_VALUE(c5 IGNORE NULLS) OVER (sparse_data) AS c5,
  LAST_VALUE(c6 IGNORE NULLS) OVER (sparse_data) AS c6,
  LAST_VALUE(c7 IGNORE NULLS) OVER (sparse_data) AS c7,
  LAST_VALUE(c8 IGNORE NULLS) OVER (sparse_data) AS c8,
  LAST_VALUE(c9 IGNORE NULLS) OVER (sparse_data) AS c9
FROM
  `testing.sparsedata`
WINDOW sparse_data AS (
  PARTITION BY id
  ORDER BY ts ASC
  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

# 结论

使用分析函数可以实现类似的效果，如 ARRAY_AGG，包括填充稀疏数据。