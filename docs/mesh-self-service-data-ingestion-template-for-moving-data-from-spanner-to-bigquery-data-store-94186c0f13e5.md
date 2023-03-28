# 数据网格自助服务—从 Spanner 到 BigQuery 的摄取模式(网格的数据存储)

> 原文：<https://medium.com/google-cloud/mesh-self-service-data-ingestion-template-for-moving-data-from-spanner-to-bigquery-data-store-94186c0f13e5?source=collection_archive---------4----------------------->

创建和管理数据源的集中一切的方法通常被认为是将其集成到业务分析堆栈中的瓶颈。随着未来技术的进步，跨生态系统摄取、存储、处理和访问更多数据变得更加容易，而无需花费大量成本。

这种技术进步需要辅之以管理数据的组织方法的相应变化。

数据网格是一种高级体系结构，它解决了一切都集中的方法的缺点，从组织的角度来看，这种方法不容易扩展和补充。

![](img/1aef1cbef0ab5bba11290bcfe2cec50e.png)

约翰·施诺布里希在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上的照片

**数据网格原理**

1.  分散的面向领域的所有权
2.  数据作为一种产品
3.  自助式数据平台
4.  联合治理

在本系列的前一部分“使用模板在数据网格中构建数据产品”中，我们讨论了使用模板作为在网格中构建和发布数据产品的自助服务的一个方面。

**网格的数据存储**

应用程序的事务存储可以是任何存储系统，如 SQL 数据库、NoSQL 数据库、文件存储或对象存储。为了将这些数据用于任何分析目的，必须将这些数据接收到适合运行分析工作负载的存储中，例如数据仓库或数据湖。

网状架构中的数据产品构建在适合分析工作负载的数据存储之上。在 Google 云平台上构建网格时，我们通常使用 BigQuery 作为网格的数据存储。

在本系列的这一部分中，我们将讨论一个这样的场景，并讨论 Spanner to BigQuery 数据摄取模式以及这种模式的自助服务模板，不同的域可以使用该模板在 BigQuery 中构建数据产品。

**摄取模式—扳手转大查询**

从 Spanner 到 BigQuery 的数据接收可以通过以下三种方式之一执行:

1.  **批量模式** — *定期从 Spanner 到 BigQuery 之间的数据批量加载*
2.  **流式模式** — *将扳手的变化数据近实时流式传输到 big query*
3.  **并联模式** — *批量负载和流式 CDC 负载的组合*

让我们探讨一下上述选项的优缺点，并确定将 Spanner 转化为 BigQuery 摄取模式的最佳方法。

**批处理模式——从 Spanner 向 BigQuery 批量加载数据**

将数据从 Spanner 数据库中的表加载到 BigQuery 数据集的一种非常简单但效率不高的方法是在一次加载操作中执行数据的完整拷贝批量传输。

这可以通过对 BigQuery 使用[云扳手联邦查询](https://cloud.google.com/bigquery/docs/cloud-spanner-federated-queries)来实现。它的实施需要两个步骤:

1.  查询 Spanner information_schema 视图以读取表模式并在 BigQuery 中创建一个表。
2.  使用联邦查询读取扳手表数据，并将其插入 BigQuery 表中

*以下示例返回关于表* `*MyTable*` *:* 中的列的信息

```
SELECT *
FROM EXTERNAL_QUERY(
  'my-project.us.example-db',
  '''SELECT t.column_name, t.spanner_type, t.is_nullable
    FROM information_schema.columns AS t
    WHERE
      t.table_catalog = ''
      AND t.table_schema = ''
     AND t.table_name = 'MyTable'
    ORDER BY t.ordinal_position
  ''');
```

*下面的示例对名为* `*orders*` *的扳手数据库进行联合查询，并将结果与名为* `*mydataset.customers*` *的 BigQuery 表连接。*

```
SELECT c.customer_id, c.name, rq.first_order_date
FROM mydataset.customers AS c
LEFT OUTER JOIN EXTERNAL_QUERY(
  'my-project.us.example-db',
  '''SELECT customer_id, MIN(order_date) AS first_order_date
  FROM orders
  GROUP BY customer_id''') AS rq 
  ON rq.customer_id = c.customer_id
GROUP BY c.customer_id, c.name, rq.first_order_date;
```

这种方法需要通过 IaC 模板部署以下资源之一:

*   具有脚本的云功能，利用 BigQuery 和外部连接 API 以及 SQL 查询进行数据加载
*   云与容器化的 web 应用一起运行，在内部利用 BigQuery 和外部连接 API 以及 SQL 查询进行数据加载
*   用于编排 BigQuery 和外部连接 API 以及数据加载任务的 Cloud Composer 或云工作流

这种方法的 IaC 模板将根据从上述选项中选择的服务而变化。

优点:

*   每次加载全部数据实现起来非常简单。
*   这种方法对于增量数据标识不依赖于特定的时间戳字段。

缺点:

*   对于 Cloud Spanner 中的大型表，每次加载完整的数据可能会很慢并且成本很高。
*   在满负荷运行期间，需要暂停对 Spanner 的写操作；否则，满载期间的写入可能会丢失。

**流模式—流扳手 CDC 到 Bigquery**

将云 Spanner 数据提取到 BigQuery 的另一种方法是从 CRUD 操作中捕获更改数据，即 Spanner 数据库中表的插入、更新和删除，并将它们传输到 BigQuery 表。

来自扳手的更改数据可以通过使用扳手更改流来捕获。更改流近乎实时地观察和流出云扳手数据库的数据更改—插入、更新和删除。关于扳手更换流程的更多信息可在[此处](https://cloud.google.com/spanner/docs/change-streams)探究，以及设置和配置程序。

一旦配置了变更流，数据变更记录就会被变更流捕获。这些数据更改记录可以传输到 BigQuery，以便在 Spanner 和 BigQuery 表之间加载和同步数据。这可以通过使用数据流与谷歌提供的模板'[云扳手改变流大查询](https://cloud.google.com/dataflow/docs/guides/templates/provided-streaming#cloud-spanner-change-streams-to-bigquery)'来实现。

Cloud Spanner change streams to BigQuery 模板是一个流管道，它将 Cloud Spanner 数据更改记录进行流式处理，并使用数据流运行器 V2 将它们写入 big query 表中。

这种方法需要通过 IaC 模板部署以下所有资源:

*   云扳手实例和数据库，表和扳手通过指定 [DDL 查询](https://cloud.google.com/spanner/docs/change-streams/manage#create)改变流。
*   使用 google 提供的模板' [Cloud Spanner 将数据流更改为 BigQuery](https://cloud.google.com/dataflow/docs/guides/templates/provided-streaming#cloud-spanner-change-streams-to-bigquery) '的云数据流作业。
*   BigQuery 数据集将用作数据流传输作业传输的数据更改记录的目标。这些表将由数据流作业自动创建(在模板中实现)。

优点:

*   这不需要任何查询联合和外部连接。
*   来自 spanner 的数据更改记录以近乎实时的方式传输到 BigQuery。
*   这种方法与 spanner 中记录的数量和表的大小无关，并且易于扩展。

缺点:

*   这将仅捕获从启用扳手更改流的时间点开始的更改。历史数据无法传输到 Bigquery。

并行模式—批量加载和流式 CDC 加载的组合

考虑到上述两种方法的优点和缺点，我们可以通过组合这两种方法来消除每个单独方法的缺点，即，组合批处理全加载和流式 CDC 加载来从 Spanner 获取数据并加载到 BigQuery。这将包括使用联邦查询从 Spanner 到 BigQuery 的一次性完整数据加载的并行化。

解决缺点:

*   由于这是一次性负载，而不是重复的全负载，因此它只会产生一次时间和成本，即第一个全负载周期。
*   启用与历史加载并行的更改流，历史加载期间的后续写入也将被并行捕获，从而解决了在全加载期间丢失数据记录的问题。

用于这种方法的 IaC 模板将包括两个同时发生的子管道，批量满载和流式 CDC 装载。这实质上是第一种和第二种方法的 IaC 与一个公共主配置文件的组合。

部署和执行的顺序如下:

1.  启用扳手更换流。
2.  为满负荷部署批处理管道。
3.  为 CDC 负载部署流式管道。一旦配置了数据流作业，CDC 加载将会自动启动。
4.  启动历史加载的批处理管道(以确保捕获更改并与满负荷并行加载)。

优点:

*   在 BigQuery 中只提取新变化并与现有数据合并的有效方法
*   易于复制高吞吐量的大型表(每秒写入大量插入或更新)，复制延迟低到中等；从长远来看接近实时。

**摘要** 在本系列的前一部分中，[使用模板在数据网格中构建数据产品](/google-cloud/build-data-product-in-data-mesh-using-templates-e1bf5a38bf86)，我们讨论了使用模板作为在网格中构建和发布数据产品的自助服务的一个方面。

在这一部分中，我们讨论了 Spanner to BigQuery 模式对摄取管道模板的一个这样的需求、不同的数据加载方法以及一些脚本和 IaC 模板片段，它们可以用作构建 mesh 摄取功能的基础。

在本系列博客的后续部分中，我们将讨论更多的摄取管道，例如将数据从云 SQL 移动到 BigQuery，将云存储中的文本文件移动到 BigQuery，将 Pub/Sub 移动到 BigQuery 和/或将 Apache Kafka 移动到 Bigquery。

参考资料:

*   [使用联邦查询从 BigQuery 查询云扳手中的数据](https://cloud.google.com/bigquery/docs/cloud-spanner-federated-queries)
*   [扳手更改流—将数据更改流传送到其他应用程序](https://cloud.google.com/spanner/docs/change-streams)
*   [使用 Terraform 创建 IaC 模板的指南和建议](https://cloud.google.com/docs/terraform/best-practices-for-terraform)