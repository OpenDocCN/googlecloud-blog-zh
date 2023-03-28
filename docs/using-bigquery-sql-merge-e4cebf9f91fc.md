# 使用 BigQuery SQL 合并

> 原文：<https://medium.com/google-cloud/using-bigquery-sql-merge-e4cebf9f91fc?source=collection_archive---------0----------------------->

BigQuery 可以完成一些非常复杂的数据处理，但是最好的特性往往隐藏在文档深处。

在这个博客中，我们将转换和合并输入数据和一些排序数据。我们将逐步构建 SQL，并使用其中的一些特性。这可以帮助你展示什么是可能的！

> ***想自己跑？请参见最后的“测试”!***

# 挑战

我们用这种结构存储我们的 **staging** 事务表:

```
STRUCT<
  id STRING,          -- globally unique
  version TIMESTAMP,  -- transaction version, greatest is current
  ... other fields (eg, customer name, product name, order id)
>
```

事务通常是唯一的(只有一个版本)，但有时会有多个版本。对于我们的分析，我们想要分析正常情况下每个事务的最新版本。我们不关心事务中的所有数据字段——不管怎样，您将看到 SQL 是如何工作的！

我们用以下结构存储我们的**分析**事务表:

```
STRUCT<
  id STRING, -- transaction with most recent version
  latest STRUCT<version TIMESTAMP, ... other fields>, -- older transactions (if any) sorted newest to oldest
  history ARRAY<STRUCT<version TIMESTAMP, ... other fields>>
>
```

这为我们提供了一种方便的方法来分析每个事务的最新的事务，同时如果我们需要历史记录的话，还可以使用 big query array 工具。

暂存数据在**transactions . staging _ data**表中，分析表在 **transactions.data** 表中。我们将构建一个 BigQuery SQL 来将 **staging_data** 表合并到 **data** 表中。该 SQL 可以多次运行而不会产生影响。

# SQL

SQL 将被编写为好像它将被划分到不同的表中，但是它可以被合并到一个单独的 SQL 中，并以 with 结尾。

## GroupedStagingTransactions

使用 [ARRAY_AGG](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#array_agg) 根据 id 聚合暂存交易。注意使用“数据”将整行作为一个结构来引用。

```
SELECT
  id,
  ARRAY_AGG(data) AS trans
FROM
  transactions.staging_data data
GROUP BY
  1
```

## 分组交易

使用[选择修饰符](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#select-modifiers) EXCEPT 和 AS 结构、[数组](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#array)函数和[表达式子查询](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#expression-subqueries)删除 ID 列。

```
SELECT
  id,
  ARRAY(SELECT AS STRUCT
      * EXCEPT (id)
    FROM
      d.trans
  ) AS trans
FROM
  GroupedStagingTransactions d
```

## 加入交易

使用 [ARRAY_CONCAT](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#array_concat) 和一个[数组构造](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays#constructing-arrays)来连接现有数据。

```
SELECT
  staging.id,
  IF(data IS NULL,
    staging.trans,
    ARRAY_CONCAT(staging.trans, [data.latest], data.history)
  ) AS trans
FROM
  GroupedTransactions AS staging
  LEFT JOIN transactions.data AS data
  ON staging.id = data.id
```

## 分类交易

使用更多的[表达式子查询](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#expression-subqueries)和一个 SELECT DISTINCT 和 SELECT AS 结构来提取不同历史记录的数组。

观察子查询中的子查询以及与**分析**表相同的输出。

```
SELECT
  t.id,
  (SELECT
     rec
   FROM
     t.trans AS rec
   ORDER BY
     version DESC
   LIMIT 1
  ) AS latest,
  ARRAY(SELECT DISTINCT AS STRUCT
     *
   FROM
     T.trans
   WHERE
     version < (SELECT MAX(version) FROM t.trans)
   ORDER BY
     version DESC
  ) AS history
FROM
  JoinedTransactions t
```

## 合并数据

在主表中更新或插入数据。使用顶层结构值来简化值的更新/插入。

```
MERGE transactions.data
USING SortedTransactions staging
ON
  staging.id = data.id
WHEN MATCHED THEN
  UPDATE SET
    latest = staging.latest,
    history = staging.history
WHEN NOT MATCHED BY TARGET THEN
  INSERT (
    id,
    latest,
    history)
  VALUES (
    staging.id,
    staging.latest,
    staging.history)
```

## 单个 SQL 语句

使用 WITH 将所有 SQL 放在一条语句中。

```
MERGE transactions.data
USING (
  WITH
  GroupedStagingTransactions AS (
    SELECT
      id,
      ARRAY_AGG(data) AS trans
    FROM
      transactions.staging_data data
    GROUP BY
      1
  ),
  GroupedTransactions AS (
    SELECT
      id,
      ARRAY(SELECT AS STRUCT
          * EXCEPT (id)
        FROM
          d.trans
      ) AS trans
    FROM
      GroupedStagingTransactions d
  ),
  JoinedTransactions AS (
    SELECT
      staging.id,
      IF(data.id IS NULL,
        staging.trans,
        ARRAY_CONCAT(staging.trans, [data.latest], data.history)
      ) AS trans
    FROM
      GroupedTransactions AS staging
      LEFT JOIN transactions.data AS data
      ON staging.id = data.id
  )
  SELECT
    t.id,
    (SELECT
      rec
     FROM
       t.trans AS rec
     ORDER BY
       version DESC
     LIMIT 1
    ) AS latest,
    ARRAY(SELECT DISTINCT AS STRUCT
       *
     FROM
       T.trans
     WHERE
       version < (SELECT MAX(version) FROM t.trans)
     ORDER BY
       version DESC
    ) AS history
  FROM
    JoinedTransactions t
) AS staging
ON
  staging.id = data.id
WHEN MATCHED THEN
  UPDATE SET
    latest = staging.latest,
    history = staging.history
WHEN NOT MATCHED BY TARGET THEN
  INSERT (
    id,
    latest,
    history)
  VALUES (
    staging.id,
    staging.latest,
    staging.history)
```

# 测试它

用一些生成的数据创建**transactions . staging _ data**表。使用 [GENERATE_ARRAY](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays#using-generated-values) 、 [UNNEST](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#from-clause) 和 [RAND](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-and-operators#rand) 进行帮助。

```
CREATE TEMP FUNCTION GetNewTransaction(id STRING)
AS (STRUCT(id,CURRENT_TIMESTAMP() AS version,
  TRUE AS active,
  CONCAT('customer-', CAST(FLOOR(RAND()*100) AS STRING))
    AS customer_name,
  CONCAT('product', CAST(FLOOR(RAND()*100) AS STRING))
    AS product_name,
  CAST(FLOOR(RAND() * 10000)/100.0 AS NUMERIC)
    AS amount
));CREATE OR REPLACE TABLE transactions.staging_data AS
SELECT
  GetNewTransaction(CAST(r AS STRING)).*
FROM
  UNNEST(GENERATE_ARRAY(1, 100000)) AS r;
```

从 **staging_data** 模式创建 **transactions.data** 表(空)。限制 0 是一个很好的方法来做到这一点！

```
CREATE OR REPLACE TABLE transactions.data AS
WITH TransactionWithoutKey AS (
 SELECT
   * EXCEPT (id)
 FROM
   transactions.staging_data
 LIMIT 0
)
SELECT
  staging_data.id,
  t AS latest,
  [t] AS history
FROM
  transactions.staging_data
  CROSS JOIN TransactionWithoutKey t
LIMIT
  0;
```

# 最后的想法

有几样东西可以扩展—

*   **分区。**随着数据量的增加，性能会降低。使用日期分区或集群分区会有所帮助。
*   **删节。**没有支持删除的机制。

我希望这能让您了解 BigQuery 中一些强大的技术。