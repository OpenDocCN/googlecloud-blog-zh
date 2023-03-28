# 将云扳手|复制中的流更改为 BigQuery

> 原文：<https://medium.com/google-cloud/change-streams-in-cloud-spanner-replication-to-bigquery-61b20b78118a?source=collection_archive---------0----------------------->

**谷歌云扳手**

![](img/1068542b4acd41ebfc53e36869f2e57a.png)

由[克劳迪奥·施瓦茨](https://unsplash.com/@purzlbaum?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)在 [Unsplash](https://unsplash.com/s/photos/data-analysis?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上拍摄的照片

Cloud Spanner 是 Google 高度可扩展和完全托管的关系数据库服务。它是传统数据库结构的复制品，结合了水平扩展、ACID 事务和 SQL 语义的附加功能。它还提供了跨多个区域的强大一致性，并且仍然可以在低延迟的情况下使用。它的可用性 SLA 为 99.999%，将每月停机时间缩短至 23 秒。简而言之，这是一个不折不扣的数据库。最初，Cloud Spanner 没有捕捉数据更改的功能，因此，为了克服这一缺陷，Google 现在引入了 Cloud Spanner 更改流。借助这一新功能，用户可以从 Spanner 到不同的云服务观察和传输近乎实时的数据变化。这里，我们将专门讨论将 spanner 数据更改复制到 BigQuery 以用于分析目的。

**读取变更流的方法**

*   通过数据流—使用 Apache Beam SpannerIO 连接器，这是读取变更流的推荐方法。谷歌还提供了数据流模板，为常见用例构建数据流管道，如将数据流更改到 BigQuery 和谷歌云存储。我们可以预期在不久的将来会出现更多这样的模板来集成不同的服务。在本文中，我们将讨论这种方法。
*   通过 API —使用变更流查询 API。该方法允许灵活地编写使用 Spanner API 读取变更流的定制代码/查询。必须使用 ExecuteStreamingSql API 执行查询。一个特殊的表值函数(TVF)会随着变更流自动创建。这提供了对更改流记录的访问，并且对返回结果集的大小没有限制。

**技术架构**

![](img/1270f02e3a184afafd9250ad5c14d8cd.png)

图 1:架构流程

**先决条件**

*   使用数据库和表在 Cloud Spanner 中创建一个实例
*   为读取而创建的更改流
*   在 BigQuery 中创建的数据集
*   必须创建一个服务帐户，并授予其以下 IAM 权限

1.  访问数据库、表格和索引(spanner.databases.select)
2.  用于模式更改(spanner.databases.updateDdl)
3.  对 GCS 存储桶的读/写权限(存储管理员角色)
4.  启动和执行数据流作业(数据流工作者)
5.  对 BigQuery 数据集(bigquery.dataEditor)的读/写/查询权限。

**创建变更流**

第一步是创建一个变更流，假设已经在 Cloud Spanner 中创建了数据库和表。Spanner 像对待其他模式对象一样对待变更流，因此可以使用 DDL 语句对其执行创建、修改、挂起和删除等操作。在创建变更流的同时，我们还可以灵活地跟踪各种选项的变更。它可以针对特定的表或特定的列，也可以针对数据库中的整个对象。我们可以在一个数据库中有一个或多个变更流。我们还可以选择为特定变更流所跟踪的数据定义保留期。该值的范围从最小 24 小时/1 天到最大 7 天。默认情况下，更改流将 24 小时作为 retention_period。一旦创建了变更流，您就可以返回到数据库概述，并检查变更流正在跟踪哪些对象。

![](img/f9b364802a15a88259bf454da023cc1e.png)

图 2:创建变更流

![](img/14b9cf2e50eec5eb2a4aea445b683adf.png)

图 3:扳手数据库概述

**创建数据流管道**

第二步是用现有模板创建一个数据流作业。目前，我们只有两个变更流模板。

![](img/aa18fd16c4efbf2c7e1cce5e1f071370.png)

图 4:创建数据流作业

![](img/db7bed2b2af6c179bdb5ff4338a350d3.png)

图 5:作业模板参数

一旦创建了作业，需要一段时间才能达到运行状态，因为数据流作业在 apache beam 上运行。资源按需从头开始旋转，并在作业完成后删除。

![](img/814ab59985fc3100818671e9cfd440a4.png)

图 6:工作概述

一旦作业开始运行，我们就可以从作业页面跟踪作业图表、执行细节

![](img/e9d0100d12f697e6935d865f4dd5ad49.png)

图 7:工作图表

![](img/2fcbc22203b618857ecf78ccbf8efabe.png)

图 8 工作图表

**数据变更—插入**

各种 DML 操作都可以通过控制台执行。从 insert 操作开始，该操作已成功地将数据推入现有表中(student)。

![](img/f423afe31e62a5c47440b087055420fa.png)

图 9:扳手中的插入操作

处理完数据后，在 BigQuery 中自动创建相应的表，命名约定为 **tablename_changelog** 。在本例中，它是 student_changelog。管道具有这样的功能，它将根据 BigQuery 映射数据类型，然后复制 Cloud Spanner 中定义的模式。还创建了一些附加字段作为元数据表的一部分，元数据表在处理数据时自行创建。

![](img/567040dd6e77625d5b5dfabab1ce7225.png)

图 10:大查询表

Cloud Spanner 中插入的数据被捕获并显示在 BigQuery 表中。为了验证获取的操作的类型，我们可以检查**_ metadata _ spanner _ mod _ type _**列

![](img/8f8c9ecdfd43a2cee4a783df924d08fc.png)

图 11:big query 表中插入的记录

**数据变更—更新**

更新云扳手中的一个或多个字段记录

![](img/1d837f8c550e64442f440be605d72938.png)

图 12:更新扳手中的记录

被更新的字段可以在 BigQuery 表中看到，带有更新的记录，而不改变其他列的值。尽管可以使用变更流来跟踪更新，但是我们必须意识到，在 BigQuery 中，现有的记录不会被更新。相反，一条新记录被插入到表中，所有更新和 **mod_type** 值被更新。

![](img/2b6a9777e8f85c654e25ba3eaf46a68e.png)

图 13:big query 表中更新的记录

**数据变更—删除**

删除云扳手中的一条或多条字段记录

![](img/5733a6f1d931ce6722f72f645966c52c.png)

图 14:删除扳手中的记录

数据被删除，并在 BigQuery 表中显示为空记录

![](img/69916aa6b85bcd41754e8e87d9ff40df.png)

图 15:在 Bigquery 表中作为 null 输入的已删除记录

**数据变更—添加表**

在流作业仍在运行时添加一个表，该表显示在 Spanner 中，由于更改流设置为“全部”，它会自动开始监视创建的附加表。

![](img/a8bcb223c960fd6035b1c2993242d74d.png)

图 16:在扳手中添加新表

![](img/735190409825964270b321f6e003f29a.png)

图 17:被变更流监视的添加的表

我们还可以对创建的附加表执行 DDL 操作。

![](img/daf9e07adcc5aca9eeb578ca86ac3c2d.png)

图 18:在新表上执行 DDL

即使数据被插入到 Cloud Spanner 的表中，它也无法到达 BigQuery，并作为失败的记录被复制到云存储桶中。发生这种情况的原因是，当管道运行时，模板不具备捕获在架构级别或表级别发生的更改的功能。要捕获任何其他更改，必须创建新的管道。

注意:一旦建立了管道，存储桶就会自动创建，以存储临时数据、暂存数据和失败数据。

![](img/91967f0ed429f099ec264f91f2ae6e53.png)

图 19 : GCS 存储桶捕获失败记录

![](img/5444fa3e0fad57c26d443fd6bbf3f6a6.png)

图 20:JSON 格式的失败记录

**数据更改—从现有表中删除列**

在 Cloud Spanner 中修改现有表的模式。

![](img/aa91a8aff2b54a7fe8912e0be7e68fb0.png)

图 21:表列上的 DDL 操作

![](img/b37dfe7ab4fe31cc44450c88c298259a.png)

图 22:删除一列

修改的模式反映在扳手中，但是变更流无法捕获它。在 BigQuery 中，既不会改变表模式，也不会创建具有已更改模式的新表。

![](img/d16ec47cc1e7cde0cd480c91a7c7c68d.png)

图 23:删除列后更新的模式

**变更流的限制**

*   每个数据库只能创建 10 个变更流。
*   像 JSON 和 STRUCT 这样的字段类型无法复制到 BigQuery 中。
*   如果定义为“all ”,它可以监视表和列发生的所有变化。
*   它只能捕获由于在创建更改流之前对定义的对象执行插入、更新和删除操作而发生的更改。
*   它只能监视用户创建的列和表的数据更改。
*   它不捕获发生在数据库级别的任何修改，例如添加/删除表、向现有表添加/删除列。
*   一个变更流不会观察其他的变更流。
*   到目前为止，我们在数据流中只有两个用于变更流的模板。

**结论**

更改流支持数据无缝流入 BigQuery，以确保来自 Cloud Spanner 的最新数据可用于分析和其他下游用例。

请关注此空间，获取更多此类文章。欢迎任何意见或反馈。

我要特别感谢 Sekhar Mandapati，Sagar Chavan 鼓励我完成这件作品，也感谢 T2 Sushant Nigudkar 在遇到阻碍时给予我的支持。

参考资料:[https://cloud . Google . com/blog/products/databases/track-and-integrate-change-data-with-spanner-change-streams](https://cloud.google.com/blog/products/databases/track-and-integrate-change-data-with-spanner-change-streams)