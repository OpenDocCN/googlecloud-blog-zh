# 基于波束流的大查询划分

> 原文：<https://medium.com/google-cloud/bigquery-partitioning-with-beam-streams-97ec232a1fcc?source=collection_archive---------1----------------------->

![](img/6c5dc44c6cd7c770ded457292755ad58.png)

[https://unsplash.com/photos/rHbob_bEsSs](https://unsplash.com/photos/rHbob_bEsSs)

如果你一直在使用谷歌大查询，你就会知道它能以创纪录的速度处理大量数据。它是 GCP 最成熟的产品之一，是工程师和非技术人员了解业务数据的窗口。如果你懂 SQL，你可以使用 BigQuery。

> 本文假设你对 [BigQuery](https://cloud.google.com/bigquery/) 和 [Apache Beam](https://beam.apache.org/) ( [云数据流](https://cloud.google.com/dataflow/))有所了解

但是成本呢？您只需为查询的数据量付费(在撰写本文时，大约每万亿美元)。一般来说，你不会花很多钱做典当分析。需要注意的是，当您开始将 BigQuery 与 Tableau 等提供实时仪表板的数据可视化工具集成时。我记得有一个案例，在一个实时仪表板上，一些查询一天被触发数千次，导致成本大幅上升。

幸运的是，有一些技术可以限制查询所需的数据量。首先，从只选择你需要的**列开始，所以千万不要用星号(*)。因为 BigQuery 使用列存储系统，所以它只会遍历您选择或搜索的列，只有这些列会被计费。**

# 分区表

但是大多数时候你知道你只需要搜索你的表的一部分。当然，如果你的数据有日期成分。如果你只对过去两周感兴趣，你不需要搜索整个数据集。为此，BigQuery 引入了分区表。

分区表是被分成不同碎片的表，这使得管理数据更加容易。目前，BigQuery 只提供日分区表，这意味着数据每天都被分割成分区。

这些表有一个包含日值的伪列( *_PARTITIONTIME* )。该 day 值表示该行所在的 day 分区，这样可以很容易地限制您包含和遍历的分区集。您可以在标准的*中使用它，其中*子句和 BigQuery 将自动包含正确的分区。请看下面的 *WHERE* 子句:

![](img/7fd956aa64e8ea53d1ff45a1e417af62.png)

[https://unsplash.com/photos/24tsXm7qGQE](https://unsplash.com/photos/24tsXm7qGQE)

```
WHERE
  _PARTITIONTIME 
  BETWEEN 
  TIMESTAMP("2017-06-01")
  AND 
  TIMESTAMP("2017-06-14")
```

这将只选择相关的表，并且只计算一小部分的成本，然后执行一个遍历整个数据集的查询。查看[文档](https://cloud.google.com/bigquery/docs/partitioned-tables)了解更多信息。

# 插入数据

不过，控制数据所在的分区可能有点棘手。您可以在 *_PARTITIONTIME* 中插入一个值，而行将在正确的分区中结束，这是因为该列是一个*只读伪列*。但是如果您不做任何事情，这些行将会在当天的分区中结束。在一些流用例中，这可能已经足够了，但是对于像我这样的控制狂来说，这还不够好。

控制目标分区的方法是使用表装饰器。因此，如果您想将数据放入新年分区，您的目的地应该是 **tablename$20170101** 。[文档](https://cloud.google.com/bigquery/docs/partitioned-tables)中有几个在加载作业中这样做的例子，但是在流[光束](https://beam.apache.org/)作业中如何做到呢？

## 自动分区流

诀窍(也是一个很好的秘密)是使用 TableDestination 函数。这是您提供给 **BigQueryIO 的一个函数。用一个**写**。至**法。这并不广为人知，因为所有的例子都只提供了一个固定的输出表。

该函数的目的是将元素映射到表中。一旦 [Apache Beam](https://beam.apache.org/) 管道遇到新窗口，它将调用函数来查看表名应该是什么。

获取固定窗口，并向 BigQuery 编写器提供 TableReference 函数。

因此，当你建立你的光束管道，确保有一个窗口集，不相交的一天。我最喜欢的是 1 分钟的固定窗口，这样我们就不必等待数据到达 BigQuery。当我们查看函数时，您会看到为什么需要一个窗口，然后将函数提供给。发挥作用。

该函数执行从 ValueInSingleWindow 到 TableDestination 的转换。奇迹就发生在这里。因此，对于每个窗口，这个函数将被调用，您可以创建正确的表名。

[https://gist . github . com/alexvanboxel/0 f 38 CEB 5 ccebbcbcd 759576 be 428 c 2](https://gist.github.com/alexvanboxel/0f38ceb5ccebbccebcd759576be428c2)

有界窗口有一个有趣的方法，那就是 maxTimestamp。如果您使用它并将其截断为一天，您就有了目标分区。而且不用担心，最大时间戳真的没问题，看例子:

> max = 2017–01–01t 23:59:59.99999 > > >表名$20170101

因此，使用修饰后的(tablename$YYYYMMDD)表引用，数据将最终出现在正确的分区中。不过，在进行流式处理时，要确保表以正确的模式存在。

就这样了。现在，您可以将数据传输到正确的分区中，*不必担心*较晚的数据会到达不正确的分区。

完整的例子你可以看看 GitHub 上的**发布备份**示例报告:

[](https://github.com/alexvanboxel/pubsub-backup) [## alexvanboxel/pubsub-backup

### 通过在 GitHub 上创建一个帐户来为 pubsub-backup 开发做贡献。

github.com](https://github.com/alexvanboxel/pubsub-backup) 

它已经适应了阿帕奇梁 2.0