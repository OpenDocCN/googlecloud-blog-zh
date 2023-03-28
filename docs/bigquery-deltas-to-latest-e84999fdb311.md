# BigQuery:最新的增量

> 原文：<https://medium.com/google-cloud/bigquery-deltas-to-latest-e84999fdb311?source=collection_archive---------2----------------------->

# 挑战

一位同事向我提出了一个挑战:

> 我进行了一系列更新，只设置了更改的字段。空表示没有变化。我有原始记录。我如何在 BigQuery 中处理这个问题？

将这些放到一个 [BigQuery 脚本](https://cloud.google.com/bigquery/docs/reference/standard-sql/scripting)中会是另外一天，但是现在的焦点将是使用 BigQuery [ARRAY_AGG](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#array_agg) 来处理这些变化。

*更新—一个额外的模式被添加到*[*big query:deltas to latest—所有历史*](/@mescanne/bigquery-delta-to-latest-all-history-c8f28f889c91) *。*

# 解决方案

## 测试数据

OriginalData 将捕获所有字段的当前非空状态。在本例中，name 是主键，id 是递增的 INT64，表示“最新”和最佳值。

```
OriginalData AS (SELECT * FROM UNNEST([
  STRUCT('nameA' AS name, 0 AS id, 20 AS age, 'superA' AS type),
  STRUCT('nameB' AS name, 0 AS id, 21 AS age, 'superB' AS type),
  STRUCT('nameC' AS name, 0 AS id, 22 AS age, 'superC' AS type),
  STRUCT('nameD' AS name, 0 AS id, 23 AS age, 'superD' AS type)
]))
```

UpdatedData 表示传入的增量。始终设置姓名和 id，但年龄和类型可能为空

```
UpdatedData AS (SELECT * FROM UNNEST([
  STRUCT('nameA' AS name,
    3 AS id,
    10 AS age,
  CAST(NULL AS STRING) AS type),
      STRUCT('nameA' AS name,
    2 AS id,
    23 AS age,
    'superA-2' AS type),
  STRUCT('nameA' AS name,
    4 AS id,
    CAST(NULL AS INT64) AS age,
    CAST(NULL AS STRING) AS type)
]))
```

## 询问

将其放入一个可运行的查询中。

数据被联合在一起，所以我们有变更和原始数据，并按名称分组。

ARRAY_AGG 在逐个字段的基础上执行以下操作:

*   将所有非空字段排列成一个数组
*   按 ID 降序排列(因此第一个位置的值是最大的 ID)
*   拉出 0 处的偏移

全部在一个查询中:

```
WITH
  OriginalData AS (SELECT * FROM UNNEST([
    STRUCT('nameA' AS name, 0 AS id, 20 AS age, 'superA' AS type),
    STRUCT('nameB' AS name, 0 AS id, 21 AS age, 'superB' AS type),
    STRUCT('nameC' AS name, 0 AS id, 22 AS age, 'superC' AS type),
    STRUCT('nameD' AS name, 0 AS id, 23 AS age, 'superD' AS type)
  ])),
  UpdatedData AS (SELECT * FROM UNNEST([
    STRUCT('nameA' AS name,
      3 AS id,
      10 AS age,
      CAST(NULL AS STRING) AS type),
    STRUCT('nameA' AS name,
      2 AS id,
      23 AS age,
      'superA-2' AS type),
    STRUCT('nameA' AS name,
      4 AS id,
      CAST(NULL AS INT64) AS age,
      CAST(NULL AS STRING) AS type)
  ]))
SELECT
  name,
  MAX(id) AS id,
  ARRAY_AGG(age IGNORE NULLS ORDER BY id DESC)[OFFSET(0)] AS age,
  ARRAY_AGG(type IGNORE NULLS ORDER BY id DESC)[OFFSET(0)] AS type
FROM
(
  SELECT * FROM OriginalData
  UNION ALL
  SELECT * FROM UpdatedData
)
GROUP BY
  name;
```

# 结论

这展示了如何逐个字段地使用 ARRAY_AGG 来查找最新的非空值。

使用 BigQuery 脚本，这可以合并到将增量应用于快照并更新快照的脚本中。那是以后的事了。