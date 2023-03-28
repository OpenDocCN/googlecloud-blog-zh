# Google Cloud 的气流:第 1 部分— BigQuery

> 原文：<https://medium.com/google-cloud/airflow-for-google-cloud-part-1-d7da9a048aa4?source=collection_archive---------0----------------------->

你知道大数据…这是一个肮脏的行业。所有的文献都向您展示了所有这些数据处理器和查询引擎有多么强大，但这一切都假设所有的数据都已准备好供消费。实际上，很多自动化数据流、Spark 和 BigQuery ETL 流程都是用 bash 或 Python 粘在一起的。

好吧，是时候改变这一点了…看看[阿帕奇气流](https://airflow.incubator.apache.org/)。Airflow 是一个工作流引擎，它将确保您的所有转换、处理和查询作业都能在正确的时间、顺序以及所需数据可供使用的时间运行。不再需要编写大量脆弱的样板代码来调度、重试和等待:只需关注工作流。

![](img/0c9860fe96c2c8cac00e22acaaa9c489.png)

[https://unsplash.com/@jeisblack?photo=stLXUcN2Dac](https://unsplash.com/@jeisblack?photo=stLXUcN2Dac)

2016 年，大量工作被注入到气流中，使其成为谷歌云的一流工作流引擎。这个 Medium 系列将解释如何使用 Airflow 来自动化许多 Google Cloud 产品，并使数据从一个引擎平滑地过渡到另一个引擎。我不会深入研究如何将 DAG 构建到气流中，你应该阅读相关的文档。基本上所有的 DAG 都是通过 Python 对象构建的…我将把重点放在谷歌云集成上。

# BigQuery 集成

在第一部分中，我们将解释如何从 Airflow 中自动化 BigQuery 任务。

```
***Note***: The series talks about the upcoming **Airflow 1.8**, make sure you have the latest verion. You can now pip install airflow.
```

BigQuery 是非常流行的交互式查询大型数据集的工具，但我也喜欢用它来存储大量临时数据。比起云存储上的文件，我更喜欢它，因为你可以在它上面做一些特别的探索性查询。因此，让我们开始使用 Airflow 将数据输入和输出 BigQuery。

## 问题

第一个 BigQuery 集成是执行一个查询并将输出存储在一个新表中，这是通过`BigQueryOperator`完成的。操作符接受一个查询(或对查询文件的引用)和一个输出表。

如果你看一下`**destination_dataset_table**`，你会注意到模板参数。Airflow 使用 jinja 模板的强大功能，使您的工作流程更加动态和具有上下文意识。在我们的例子中，它将在`**ds_nodash**`中填入当前的执行日期。气流中的执行日期是数据的上下文日期。例如，如果您计算 7 月 4 日的一些指标，`**execution_date**`将是`2017–07–04T00:00:00+0`。

您甚至可以在引用的查询中使用 jinja 模板。这是一个强大的工具，因为通常 BigQuery API 不支持参数，但是使用模板引擎可以模拟这种情况。

关于什么是可能的以及什么类型的参数在上下文中可用的完整列表，请查看气流宏文档。

在我们的示例中，上述查询文件位于示例 DAG 旁边的文件系统中。

## 提取到存储

除了查询之外，您还想从 BigQuery 中获取数据。让我们从提取到云存储开始。

提取是一个非常常见的场景，用`BigQueryToCloudStorageOperator`来完成。熟悉 BigQuery API 的人会认识到许多参数，但基本上您必须指定源表和目标存储对象模式。但是指定一个模式很重要(例如:`…/part-.avro`)。该模式将确保如果您有大型表，那么每次提取都有多个存储对象。

再次注意 jinja 模板参数的使用。在这种情况下，我们读取气流中定义的变量，这对于区分不同环境非常有用，如*生产*或*测试*。

## 从仓库装载

但是在进行查询和提取之前，您需要将数据放入 BigQuery。`GoogleCloudStorageToBigQueryOperator`已经完成了。

与提取相比，加载操作符确实有更多的参数。原因是您需要告诉 BigQuery 导入对象的一些元数据，比如模式和格式。通过指定`schema_fields`和`source_format`可以做到这一点。

另一个需要指定的重要参数是`create_disposition`和`write_disposition`。我喜欢我的操作是可重复的，这样你就可以一次又一次地运行它们，所以`**CREATE_IF_NEEDED**`和`**WRITE_TRUNCATE**`是很好的默认值。只要确保您的表能够处理它，那么就使用分区表。现在，您必须使用名称中带有日期后缀的传统分区。在 Airflow 的下一个版本中，我们将添加对新的 BigQuery 分区的完全支持。

在这个系列的下一篇文章中，我们将会看到气流驱动的 Dataproc，到时候见。

示例摘自《谷歌云集成测试与示例的气流:[https://github.com/alexvanboxel/airflow-gcp-examples](https://github.com/alexvanboxel/airflow-gcp-examples)