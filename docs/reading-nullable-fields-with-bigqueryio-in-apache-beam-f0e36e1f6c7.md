# 在 Apache Beam 中使用 BigQueryIO 读取可空字段

> 原文：<https://medium.com/google-cloud/reading-nullable-fields-with-bigqueryio-in-apache-beam-f0e36e1f6c7?source=collection_archive---------0----------------------->

![](img/74742fcbbb47505a742407d73953f467.png)

照片由[本·赫尔希](https://unsplash.com/@benhershey?utm_source=medium&utm_medium=referral)在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄

当在用 Java 编写的 Apache Beam 管道中使用 [BigQueryIO](https://beam.apache.org/releases/javadoc/2.22.0/org/apache/beam/sdk/io/gcp/bigquery/BigQueryIO.html) 从 BigQuery 读取数据时，您可以将记录读取为`TableRow`(方便但性能较低)或`GenericRecord`(额外的性能，但您的代码中可能也需要一些额外的工作)。

[在之前的一篇文章](/google-cloud/reading-numeric-fields-with-bigqueryio-in-apache-beam-23273a9d0c99)中，我展示了当使用 *GenericRecord，*big query 类型或多或少容易映射到 Java 类型 NUMERIC 除外，这需要一些额外的工作。

但这实际上只适用于*必需的*big query 列。**可空的*列*需要一些额外的解析，以便从记录中提取值。**

在 BigQuery 中为表创建模式时，[我们可以为列](https://cloud.google.com/bigquery/docs/schemas#modes)提供三种不同的模式:*可空*、*必需*和*重复*。

默认模式是*可空*。可空字段可能包含该列类型的值，或者 null。简单。

然而，当将 BigQueryIO 中的数据读取为 *GenericRecord* 时，Avro 模式没有可空类型。除了其他类型之外，它们还有一个空的类型。所以要指定一个字段可以包含空值，需要在 Avro 中为同一个字段指定几种类型。那其实叫*联*。例如，如果您的 name *myfield* 字段包含字符串并且可为空，则 Avro 模式将如下所示:

```
{"name": "myfield", "type": ["null", "string"]}
```

在 *GenericRecord* 中， *myfield* 的类型将是 *union* ，我们需要遍历该字段的*类型*和*值*，以实际恢复 *myfield* 的值(空值或字符串)。

所以让我们假设我们有以下两个变量(所有代码片段都是 Java 的):

```
String fieldName;   // the name of the column
GenericRecord rec;  // the values in the row to be parsed
```

我们可以恢复该字段的模式和类型

```
Schema fieldSchema = rec.getSchema().getField(fieldName).schema();
Schema.Type type = fieldSchema.getType();
```

现在，我们应该检查类型是否是 UNION

```
if (type == Schema.Type.UNION) {      
   // We have a NULLABLE field
...
```

我们也可以检查`fieldSchema.getTypes()`的长度是 1 还是 2。对于必填字段，该值为 1；对于可空字段，该值为 2。

在我们检查了我们有一个可空字段之后，我们可以恢复类型，尝试识别空类型(在字段包含的两种类型中)，并使用另一种类型来解析值:

```
List<Schema> types = fieldSchema.getTypes();      
// Let's see which one is the NULL and let's keep the other      
if (types.get(0).getType() == Schema.Type.NULL) {        
  type = types.get(1).getType();      
} else {        
  type = types.get(0).getType();      
}
```

之后，我们可以将该值解析为另一个*所需的*值(也就是说，只有一种类型)。请注意，如果表中的原始值为 null，则该字段可能包含 null。

查看完整的代码片段:

从 BigQuery 读取时，不要放弃使用 *GenericRecord。*有时你将不得不做一些额外的解析，比如在这个例子中，当你使用数字字段时，使用 NULLABLE 或[，但是你将获得的额外性能值得多几行代码。](/google-cloud/reading-numeric-fields-with-bigqueryio-in-apache-beam-23273a9d0c99)

对于读取 NULLABLE，请记住该类型将是两种类型的联合:字段的实际类型和 null。读取值时要小心空值！