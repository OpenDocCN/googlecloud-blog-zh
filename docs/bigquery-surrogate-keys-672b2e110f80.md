# BigQuery 代理键

> 原文：<https://medium.com/google-cloud/bigquery-surrogate-keys-672b2e110f80?source=collection_archive---------0----------------------->

[代理键](https://en.wikipedia.org/wiki/Surrogate_key)是传统数据仓库中常见的概念，但在 BigData 中并不常见。生成一个连续递增的数字是很难并行化的，替代方法，如 HASH 或 CONCAT，可以提供类似的唯一标识符。

我将使用以下模式探索 BigQuery 中的一个特定场景:

*   三个键(key1、key2、key3)和创建代理键(id)的需要。
*   带有预分配代理键的传入数据和现有数据。

注意:您需要在项目中创建**代理键**作为数据集，以便 SQL 运行。

# 创建测试数据

## 输入数据

用(key1，key2，key3)和数据创建一个表**。这将具有与现有代理键部分匹配的键分布。**

```
CREATE OR REPLACE TABLE `surrogatekey.incoming_data` AS
SELECT
  -- Actual keys; need a synthetic key
  CAST(k1 AS STRING) AS key1,
  CAST(k2 AS STRING) AS key2,
  CAST(k3 AS STRING) AS key3, -- Data
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col1,
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col2,
  'x' AS col3,
  'y' AS col4
FROM
  UNNEST(GENERATE_ARRAY(5, 20)) k1
  CROSS JOIN UNNEST(GENERATE_ARRAY(5, 20)) k2
  CROSS JOIN UNNEST(GENERATE_ARRAY(5, 20)) k3;
```

## 现有数据—增加

创建一个已有预分配(递增)代理键的现有数据表。这将部分匹配输入的数据。

```
CREATE OR REPLACE TABLE `surrogatekey.existing_data` AS
SELECT
  -- Existing surrogate
  k3+(k2*10)+(k1*100) AS id, -- Actual keys; need a synthetic key
  CAST(k1 AS STRING) AS key1,
  CAST(k2 AS STRING) AS key2,
  CAST(k3 AS STRING) AS key3, -- Data
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col1,
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col2,
  'x' AS col3,
  'y' AS col4
FROM
  UNNEST(GENERATE_ARRAY(1, 10)) k1
  CROSS JOIN UNNEST(GENERATE_ARRAY(1, 10)) k2
  CROSS JOIN UNNEST(GENERATE_ARRAY(1, 10)) k3;
```

## 现有数据—哈希

创建一个现有的数据表**surrogate key . existing _ data _ hash**，但是，这一次，创建一个 HASH。这将获取结构中的键(key1，key2，key3)，使用 TO_JSON_STRING 将其转换为一个单独的字符串和 FARM_FINGERPRINT。这将产生一个 INT64 值，冲突几率极低。

```
CREATE OR REPLACE TABLE `surrogatekey.existing_data_hash` AS
SELECT
  -- Existing surrogate is a HASH
  FARM_FINGERPRINT(TO_JSON_STRING(STRUCT(
      CAST(k1 AS STRING),
      CAST(k2 AS STRING),
      CAST(k3 AS STRING)
  ))) AS id, -- Actual keys
  CAST(k1 AS STRING) AS key1,
  CAST(k2 AS STRING) AS key2,
  CAST(k3 AS STRING) AS key3, -- Data
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col1,
  CAST(FLOOR(1000 * RAND()) AS STRING) AS col2,
  'x' AS col3,
  'y' AS col4
FROM
  UNNEST(GENERATE_ARRAY(1, 10)) k1
  CROSS JOIN UNNEST(GENERATE_ARRAY(1, 10)) k2
  CROSS JOIN UNNEST(GENERATE_ARRAY(1, 10)) k3;
```

# 更新和插入数据—递增

## 处理输入数据

首先，传入的数据将被处理到**代理密钥. staging_data** 中。这将联接预先存在的代理键或分配一个新的代理键(递增)。

为了分配新的密钥，我们使用现有密钥的最大值和传入数据的递增 ROW_NUMBER()。我们按照 id 对传入的数据进行排序，因此 NULL 首先出现。

```
CREATE OR REPLACE TABLE `surrogatekey.staging_data` AS
SELECT
  e.id IS NULL AS IsNew,
  IFNULL(e.id,
    maxId + ROW_NUMBER() OVER (
      ORDER BY
        e.id
    )
  ) AS id,
  i.*
FROM
  `surrogatekey.incoming_data` i
  LEFT JOIN `surrogatekey.existing_data` e
  USING (key1, key2, key3)
  CROSS JOIN (
    SELECT
      MAX(id) AS maxId
    FROM
      `surrogatekey.existing_data`
  ) m
```

## 处理传入的数据—大数据

在前面的步骤中，为所有的 incoming_data 计算了 ROW_NUMBER()。这不是并行的，如果它占用了太多的内存，你可能会得到超过资源。我们可以修改查询，只为需要新代理键的新行计算 ROW_NUMBER()。

```
CREATE OR REPLACE TABLE `surrogatekey.staging_data` AS
WITH
  -- Join with existing surrogatekey
  JoinedData AS (
    SELECT
      e.id,
      i.*
    FROM
      `surrogatekey.incoming_data` i
      LEFT JOIN `surrogatekey.existing_data` e
      USING (key1, key2, key3)
  ),
  -- Replace NULL using ROW_NUMBER and previous max
  NewKeyedData AS (
    SELECT
      m.maxId + ROW_NUMBER() OVER () AS id,
      j.* EXCEPT (id)
    FROM
      JoinedData j
      CROSS JOIN (
        SELECT
          IFNULL(MAX(id), 0) AS maxId
        FROM
          `surrogatekey.existing_data`
      ) m
    WHERE
      j.id IS NULL
  )
-- Union together existing and new surrogate keys
SELECT
  *
FROM
  JoinedData
WHERE
  id IS NOT NULL
UNION ALL
SELECT
  *
FROM
  NewKeyedData;
```

## 处理传入的数据—较大的数据

在大数据步骤中，为需要新代理键的传入数据计算 ROW_NUMBER()。但是，所有的列都被处理了。这一次只处理关键列，并与最终数据重新连接。

```
CREATE OR REPLACE TABLE `surrogatekey.staging_data` AS
WITH
  JoinedData AS (
    SELECT
      e.id,
      i.*
    FROM
      `surrogatekey.incoming_data` i
      LEFT JOIN `surrogatekey.existing_data` e
      USING (key1, key2, key3)
  ),
  NewKeyedData AS (
    SELECT
      m.maxId + ROW_NUMBER() OVER () AS id,
      j.key1,
      j.key2,
      j.key3
    FROM
      JoinedData j
      CROSS JOIN (
        SELECT
          IFNULL(MAX(id), 0) AS maxId
        FROM
          `surrogatekey.existing_data`
      ) m
    WHERE
      j.id IS NULL
  )
SELECT
  IFNULL(j.id, n.id) AS id,
  j.* EXCEPT (id)
FROM
  JoinedData j
  LEFT JOIN NewKeyedData n
  USING (key1, key2, key3);
```

## 更新现有数据

按 id 更新现有数据。我们不需要根据它是否是新的进行过滤，因为这仅适用于连接有效的情况。

```
UPDATE `surrogatekey.existing_data` e
SET
  col1 = s.col1,
  col2 = s.col2,
  col3 = s.col3,
  col4 = s.col4
FROM
  `surrogatekey.staging_data` s
WHERE
  e.id = s.id;
```

## 插入新数据

在新行的位置插入新行。

```
INSERT INTO  `surrogatekey.existing_data`
  (id, key1, key2, key3, col1, col2, col3, col4)
SELECT
  * EXCEPT (IsNew)
FROM
  `surrogatekey.staging_data`
WHERE
  IsNew;
```

# 合并数据—递增

或者，一个 MERGE 语句根据需要生成一个新的代理键，并更新/插入到现有的表中。这是将以前的 SQL 放在一起，而不需要复杂地跟踪它是否是一个新行。

```
MERGE INTO `surrogatekey.existing_data` AS e
USING (
  SELECT
    IFNULL(e.id,
      maxId + ROW_NUMBER() OVER (
        ORDER BY
          e.id
      )
    ) AS Id,
    i.*
  FROM
    `surrogatekey.incoming_data` i
    LEFT JOIN `surrogatekey.existing_data` e
    USING (key1, key2, key3)
    CROSS JOIN (
      SELECT
        MAX(id) AS maxId
      FROM
        `surrogatekey.existing_data`
    ) m
) s
ON e.id=s.id
WHEN MATCHED THEN
  UPDATE SET
    col1 = s.col1,
    col2 = s.col2,
    col3 = s.col3,
    col4 = s.col4
WHEN NOT MATCHED THEN
  INSERT ROW
```

# 合并数据—哈希

使用散列甚至更简单——只需根据传入数据适当地更新或插入，因为散列可以由每一行计算。

```
MERGE INTO `surrogatekey.existing_data_hash` AS e
USING (
  SELECT
    FARM_FINGERPRINT(TO_JSON_STRING(STRUCT(key1, key2, key3)))
      AS id,
    *
  FROM
    `surrogatekey.incoming_data` AS i
) AS i
ON (e.id=i.id)
WHEN MATCHED THEN
  UPDATE SET
    col1 = i.col1,
    col2 = i.col2,
    col3 = i.col3,
    col4 = i.col4
WHEN NOT MATCHED THEN
  INSERT ROW
```

# 合并数据—串联

散列可能非常强大和简单，但是另一种方法是将所有的键连接在一起作为一个大字符串。这有保证唯一性的好处(只要构造得好)，并且通过查看键也是可以解释的。

使用 id 作为字符串而不是 INT64，它可以按如下方式计算:

```
CONCAT(
      CAST(key1 AS STRING), '-',
      CAST(key2 AS STRING), '-',
      CAST(key3 AS STRING)
    ) AS id,
```

# 结论

代理键可能是有用的，但是也值得考虑它们是如何被使用的以及它们引入的复杂性。虽然性能可能很重要，但查询模式可能是最重要的。

研究散列替代方案——或者更好的是，不使用代理键——是值得考虑的。