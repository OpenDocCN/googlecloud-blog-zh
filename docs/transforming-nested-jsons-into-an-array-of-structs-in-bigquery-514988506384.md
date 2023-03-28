# 在 BigQuery 中将嵌套 JSONs 转换为结构数组

> 原文：<https://medium.com/google-cloud/transforming-nested-jsons-into-an-array-of-structs-in-bigquery-514988506384?source=collection_archive---------0----------------------->

有时，您的数据作为嵌套的 JSON 字符串存放在 BigQuery 中。一个例子可能是一个 a 列，其中每一项都有一个键，关于该项的详细信息如下所示。

```
{
   "item_1": {
      "cost": 100,
      "name": "My Product",
      "category": "My Category"
   },
   "item_2": {
      "cost": 150,
      "name": "My Other Product",
      "category": "My Other Category"
   }
}
```

这种格式在 SQL 中很难使用。让我们考虑一下每个项目类别的平均成本。为了提取嵌套值，我们需要知道我们索引的实际键。例如，如果我们想要获取 item_1 的成本，我们可以利用 [JSON_EXTRACT_SCALAR](https://cloud.google.com/bigquery/docs/reference/standard-sql/json_functions#json_extract) 函数:

```
SELECT JSON_EXTRACT_SCALAR(items, '$.item_1.cost')
FROM my_table
```

但是要获取所有的类别和成本字段，我们需要知道每个条目的键，这可能很繁琐，甚至不合理，这取决于条目的数量。

```
SELECT category, AVG(cost) as average_cost
FROM
   (SELECT 
      JSON_EXTRACT_SCALAR(items, '$.item_1.category') as category,
      JSON_EXTRACT_SCALAR(items, '$.item_1.cost') as cost,
   FROM my_table
   UNION ALL
   SELECT 
      JSON_EXTRACT_SCALAR(items, '$.item_2.category') as category,
      JSON_EXTRACT_SCALAR(items, '$.item_2.cost') as cost,
   FROM my_table)
GROUP BY 1
```

我们真正想要的是重构我们的数据，这样我们就有了一个由[结构](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#struct_type)组成的[数组](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#array_type)，而不是嵌套的 JSONs。

```
[
   {
      "cost": 100,
      "name": "My Product",
      "category": "My Category" },
   {
      "cost": 150,
      "name": "My Other Product",
      "category": "My Other Category"
   }
]
```

数组代表一个重复的字段，这意味着列表中的每个对象都有点像表中自己的记录(或行)。当我们的数据被重组到一个数组中时，我们可以很容易地利用 [UNNEST](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays) 函数将每个对象分割成它自己的行——然后我们可以使用标准 SQL 查询它。

```
SELECT category, cost
FROM my_table, UNNEST(items) 
```

如果有一些共同的命名结构，我们可以通过使用带正则表达式的 [SPLIT](https://cloud.google.com/bigquery/docs/reference/standard-sql/string_functions#split) 函数将每个 JSON 项分解成数组中它自己的对象。但是这可能并不适用于所有的用例。

更好的选择可能是使用用户定义的函数来遍历 JSON 中的每个键。下面我定义了一个临时的 [Javascript UDF](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions#javascript-udf-structure) ，它获取对象中的所有键(例如 item_1，item_2)并遍历每个键来选择 JSON 并将其添加到一个数组中。

我不想为嵌套结构定义模式，所以我实际上只是将输出作为字符串数组。但是，我们可以使用 BigQuery 的 JSON 函数来获取我们感兴趣的字段并进行聚合，就像字符串本身是一个结构一样！

```
CREATE TEMP FUNCTION unnest_json(str STRING)
RETURNS ARRAY<STRING>
LANGUAGE js AS r"""
  var obj = JSON.parse(str);
  var keys = Object.keys(obj);
  var arr = [];
  for (i = 0; i < keys.length; i++) {
    arr.push(JSON.stringify(obj[keys[i]]));
  }
  return arr;
""";SELECT 
JSON_EXTRACT_SCALAR(itms,'$.category') as category,  AVERAGE(JSON_EXTRACT_SCALAR(itms,'$.cost')) as cost from my_table, UNNEST(unnest_json(items)) as itms
GROUP BY 1
```

更多 BigQuery 内容，关注我 LinkedIn 和 Twitter @leighajarett！