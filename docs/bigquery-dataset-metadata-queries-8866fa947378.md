# BigQuery 数据集元数据查询

> 原文：<https://medium.com/google-cloud/bigquery-dataset-metadata-queries-8866fa947378?source=collection_archive---------0----------------------->

![](img/cd056a99a907be80e13377a1252e4106.png)

在 BigQuery 中使用表时，您需要了解数据集结构，无论它是公共的还是您设置的，并且您想要检查它。这是一个共享查询的快捷方式，您可以使用它来提取数据集和表中的元数据。

在下面的例子中，我使用 big query public[Stack Overflow](http://console.cloud.google.com/marketplace/details/stack-exchange/stack-overflow)数据库来演示这些命令。根据需要为正在使用的数据集和表更改名称。

# 数据集元数据

获取数据集中所有表的列表以及相应的信息。

```
SELECT *
FROM bigquery-public-data.stackoverflow.INFORMATION_SCHEMA.TABLES
```

运行时查询处理了 10MB，列结果包括:

*   表 _ 目录(目录的名称)
*   table_schema(数据集名称)
*   表名
*   表格类型
*   is _ insertable _ into
*   是类型化的
*   创建时间(日期时间格式)

其他表详细信息，包括行数和表数据大小。

```
SELECT *
FROM bigquery-public-data.stackoverflow.__TABLES__
```

*注，其两条下划线在*两侧的*表上方。*

运行时处理的查询 0B 和列结果包括:

*   project_id(与模式中的 table_catalog 相同)
*   dataset_id(与模式中的 table_schema 相同)
*   table_id(与模式中的 table_name 相同)
*   创建时间(时间戳格式)
*   上次修改时间(时间戳格式)
*   行计数
*   大小 _ 字节
*   类型

在上面的查询中，我对查询进行了如下修改，使 timestamp 为 datetime 格式，size_bytes 为 GB。

```
SELECT project_id, dataset_id, table_id as table_name, CAST(TIMESTAMP_MILLIS(creation_time) AS DATETIME) as creation_time,  CAST(TIMESTAMP_MILLIS(last_modified_time) AS DATETIME) as last_modified_time, row_count, size_bytes / POW(10,9) as GB, type
FROM bigquery-public-data.stackoverflow.__TABLES__
```

我还将 *table_id* 改为 *table_name* ，以便更容易与本文中的第一个查询合并。合并时，我可以省略*项目 id* 和*表格目录*，因为它们是多余的。

# 表元数据

要获得数据集中特定表的详细信息，请从 *stackoverflow* 数据集中使用 *posts_questions* 表提取表名并包含在以下查询中，如下例所示。

```
SELECT *
FROM bigquery-public-data.stackoverflow.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = '[p](http://twitter.com/tablename)osts_questions'
```

运行时查询处理了 10MB，列结果包括:

*   表 _ 目录(目录的名称)
*   table_schema(数据集名称)
*   表名
*   列名
*   序数 _ 位置
*   is_nullable(对/错)
*   数据类型

# 费用

关于成本的另一个注意事项是，BQ 免费提供每月处理的查询数据的前 1TB。除此之外，这取决于你的定价模式。针对 INFORMATION_SCHEMA 的按需查询至少会产生 10MB 的数据处理费用，即使处理的字节数更少。对于统一费率定价，这些消耗 BQ 位。需要注意的是，这些查询不会被存储，所以每次运行查询时都会向您收费。基本上，运行元数据查询通常是名义上的。如果您想更深入地了解潜在的查询成本，请查看此 [BQ 定价资源](https://cloud.google.com/bigquery/pricing)。

# 包裹

上面回顾了从 BigQuery 中提取数据集和表元数据的几个关键查询。还有其他视图，如数据集作业、预留和例程，您可以在[信息模式指南](https://cloud.google.com/bigquery/docs/information-schema-views)中获得更多信息和详细信息。视图有助于您对正在使用的数据集和表的结构有一个全面的了解，以便您可以计划如何最好地参与。