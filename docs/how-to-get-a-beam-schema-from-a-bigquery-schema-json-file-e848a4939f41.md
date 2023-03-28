# 如何从 BigQuery 模式 JSON 文件中获取 Beam 模式

> 原文：<https://medium.com/google-cloud/how-to-get-a-beam-schema-from-a-bigquery-schema-json-file-e848a4939f41?source=collection_archive---------0----------------------->

![](img/1e3c318115d02db3ac96e914673d86fb.png)

由 [Ferenc Almasi](https://unsplash.com/@flowforfrank?utm_source=medium&utm_medium=referral) 在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄的照片

[Beam schemas](https://beam.apache.org/documentation/programming-guide/#schemas) 是一种使用 Beam SQL 和高级代码 API(例如 [joins](https://beam.apache.org/documentation/patterns/schema/#using-joins) )编写更简洁的管道的方法。

如果您用 Java 为您的数据编写一个类，那么为您的管道获得相应的模式是相对简单的。但是，如果您有一个来自 BigQuery 的 JSON 格式的表模式文件，并且希望将其作为 PCollection 的模式应用，该怎么办呢？

一种可能的情况是将动态模式指定为管道的输入参数。

例如，BigQuery JSON 模式文件如下所示:

```
{
  "id": "bigquery-public-data:github_repos.commits",
  "kind": "bigquery#table",
  "numRows": "262745001",
  "**schema**": {
    "**fields**": [
      {
        "mode": "NULLABLE",
        "name": "commit",
        "type": "STRING"
      },
      {
        "mode": "REQUIRED",
        "name": "tree",
        "type": "STRING"
      },
...
```

该文件有一个名为`schema`的字段，包含所有模式信息。该字段有另一个名为`fields`的字段，它是表模式字段的列表。

[TableSchema](https://developers.google.com/resources/api-libraries/documentation/bigquery/v2/java/latest/com/google/api/services/bigquery/model/TableSchema.html) 类是一个`GenericJson`类，它假设有一个名为`fields.`的列表字段，因此我们可以使用它来尝试解析来自 BigQuery 的任何模式，只要我们提取`schema`字段，并使用类`TableSchema`来解析该字段。

Apache Beam 已经在内部为 XML 使用了 Jackson，所以我们可以利用 Jackson 来提取`schema`字段:

```
ObjectMapper mapper = new ObjectMapper();
JsonNode topNode = mapper.readTree(schemaAsString);
JsonNode schemaRootNode = topNode.path("schema");
```

我们现在可以将`schemaRootNode`转换回 JSON 字符串，然后用它来映射到`TableSchema`

```
TableSchema tableSchema = defaultJsonFactory.fromString(
                             schemaRootNode.toString(), 
                             TableSchema.class);
```

现在，由于 BigQueryIO 的 [BigQueryUtils 模块，我们可以简单地将它转换成一个 Beam 模式:](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/io/gcp/bigquery/BigQueryUtils.html#fromTableSchema-com.google.api.services.bigquery.model.TableSchema-)

```
Schema schema = BigQueryUtils.fromTableSchema(tableSchema);
```

然后，您可以在管道中使用这个 Beam 模式，[例如，使用动态模式解析 JSONs(使用 Beam Row 类):](https://cloud.google.com/architecture/e-commerce/patterns/converting-json)

```
ParseResult parsedJson = jsonStrings.apply("Parse JSON",
    JsonToRow.*withExceptionReporting*(schema));
```

[解析 BigQuery 模式的完整代码，包括导入的所有细节，可以在 Github 的 Gist 中找到。](https://gist.github.com/iht/2a952c9a39934db0688a5e46aef9f43f)

现在，您可以使用 BigQuery 对您的模式进行编码，并在管道中使用它通过动态模式加载数据。我喜欢使用 BigQuery 的[创建表向导以可视化的方式指定复杂的模式(包括列表、嵌套字段等)，创建一个虚拟表，将该模式导出为 JSON 文件，并在我的 Beam 管道中使用它来支持动态模式。](https://cloud.google.com/bigquery/docs/tables)