# 在 BigQuery 中提取 JSON 键值对

> 原文：<https://medium.com/google-cloud/extracting-json-key-value-pairs-in-bigquery-1bb9d0ec0b6d?source=collection_archive---------0----------------------->

# BigQuery 中 JSON 的简短介绍

Bigquery 不久前引入了处理 JSON 结构的能力。它既支持表示 JSON 结构的字符串值，也支持本地 JSON 数据类型。你可以在这里找到很好的介绍[。](https://cloud.google.com/bigquery/docs/reference/standard-sql/json-data)

Bigquery 提供了使用如下路径语句查询 JSON(以及带有 JSON 的字符串)的能力:

```
JSON_EXTRACT(json_field, "$.path.to.my.data") 
```

或简称:

```
jsonfield.path.to.my.data
```

从 JSON 中提取数据有很多方便的函数，但是也有一些限制。

# 挑战

最近我遇到了一个情况，我们需要在 json 字典中提取条目。

```
{
   "temp": 30, "elevation":120
}
{
   "Ambience": "high", "sound":"fair"
}
{
    "temp": 10, "sound":"bad"
}
```

最简单的方法是使用:

```
JSON_EXTRACT(json_expr, "$.temp")
```

不幸的是，运行时密钥是未知的，所以不可能编写 json 路径。

第一种方法从可能的关键字列表开始:

```
DECLARE wanted_keys array<STRING> 
   DEFAULT  ARRAY["temp","Ambience", "sound"];
```

然后可以迭代`wanted_keys`来提取值

```
SELECT  
   city_record.city, 
   wanted_key.name,
  JSON_EXTRACT(city_record.payload, '$.'||wanted_key.name))
```

事实证明，我们不能在`JSON_EXTRACT`函数中动态创建路径参数。它必须是常量或查询参数(即它需要是常量)。

因此，我们引入了一个定制的 Java 脚本来提取基于动态键的值:

```
CREATE temp  FUNCTION CUSTOM_JSON_EXTRACT(json STRING, json_path STRING) RETURNS STRING
LANGUAGE js AS """ 
  try {
     var obj = JSON.parse(json);
     var result;
     var arg;
     if (obj) {
        if (json_path && obj.hasOwnProperty(json_path)) {
          result = JSON.stringify(obj[json_path]);
        }
        return (result.length ? result : false);
     }
} catch (e) { return null }
""";
```

该查询现在可以遍历数据记录和所需的键，并一次提取一行:

```
FOR city_record in (SELECT  city, payload from DATA)
DO
  FOR wanted_key in (SELECT * from unnest(wanted_keys) as name)
  DO
    IF city_record.payload like '%"'||wanted_key.name||'"%'
    THEN
      INSERT INTO results (
        SELECT  city_record.city, wanted_key.name,
        CUSTOM_JSON_EXTRACT(city_record.payload, wanted_key.name));
     END IF;
   END FOR;
END FOR;
```

这样做很好，但是有一个明显的问题:这个双循环中的每个`INSERT INTO results`语句都是一个 Bigquery 任务！如果您处理 1000 行 3 个可能的键，您将启动多达 3000 个 bigquery 作业。它们将按顺序执行。Bigquery 所有的并行处理能力都消失了！

# 更好的解决方案

更好的解决方案是:

*   创建一个 javascript，将字典转换成键值对列表
*   将它们与原始行交叉连接，为每个键创建一行
*   用`wanted_keys`列表连接这些行

```
-- extract all key value pairs as an array from a json dict
-- input: json string with a dictionary
-- returns:  list of struct <key, value>
CREATE TEMP  FUNCTION EXTRACT_KV_PAIRS(json_str STRING)
RETURNS ARRAY<STRUCT<key STRING, value STRING>>
LANGUAGE js AS """
  try{ 
    const json_dict = JSON.parse(json_str); 
    const all_kv = Object.entries(json_dict).map(
        (r)=>Object.fromEntries([["key", r[0]],["value",  
                                   JSON.stringify(r[1])]]));
    return all_kv;
  } catch(e) { return [{"key": "error","value": e}];}
""";-- make our sample data
WITH
make_table AS(
   SELECT "NYC" as City, '{"temp": 30, "elevation":120}' as payload
   UNION ALL
   SELECT "SF" ,  '{"Ambience": "high", "sound":"fair"}'
   UNION ALL
   SELECT "LA" ,  '{"temp": 10, "sound":"medium"}'
),
-- make the sample wanted keys
wanted_keys AS(
  SELECT * FROM
  UNNEST(ARRAY["temp","Ambience", "sound"]) 
  AS wanted
),
-- turn the dictionary into a list
extracted_pairs as (
  SELECT city,
  EXTRACT_KV_PAIRS(payload) as kv_list 
  FROM make_table
),
-- now expand the list into individual rows
all_pairs AS(
  SELECT city, kv_pair.key AS key, kv_pair.value AS value
  FROM extracted_pairs
  CROSS JOIN UNNEST (kv_list) as kv_pair
)
-- now restrict it to those rows 
-- that match the keys we are looking for
SELECT city, key, value
FROM all_pairs
JOIN wanted_keys on key = wanted
```

这个解决方案以全 BigQuery 速度运行。以前的解决方案需要几分钟甚至几小时才能运行，而现在的解决方案只需几秒钟就能运行