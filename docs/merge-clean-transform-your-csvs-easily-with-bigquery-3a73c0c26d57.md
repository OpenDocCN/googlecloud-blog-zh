# 使用 BigQuery 轻松合并、清理和转换您的 CSV

> 原文：<https://medium.com/google-cloud/merge-clean-transform-your-csvs-easily-with-bigquery-3a73c0c26d57?source=collection_archive---------1----------------------->

![](img/098454327098af8c198af9e13026126d.png)

CSV(逗号分隔值)格式是**最常见的文本文件格式**之一，用于共享结构化数据，易于**人类**，任何软件如 **Google Sheet 和 Microsoft Excel** ，或数据库引擎如 **MySQL、PostgreSql 或 BigQuery** 阅读。

然而，这种格式的流行导致**将它用于大量数据**难以在本地管理**并将它们加载到软件或本地数据库中。因此，通过保持 CSV 格式，一些**基本操作变得无法**手动快速执行:**

*   **更新 CSV 文件内容**，如清除不良值或删除过时行。
*   **更新字段格式**，如日期格式，或单位。
*   **合并或拆分列**，如地址。
*   将几个文件连接在一起，尤其是当你有一个支点的时候。
    ***例如*** ，一个客户的 CSV 文件(带`customerId`)和一个订单的 CSV 文件(每个订单带`customerId`),你要在`customerId`上加入这些文件以获得每个订单的客户详细信息。

*在 Google Cloud 上，你有，大部分时间，* ***这些 CSV 文件都存储在云存储上。***

# 计算引擎(坏)解决方案

如果您的**本地机器不够强大**来管理 CSV 文件，**第一个想法是在计算引擎上使用强大的云服务器**。

该过程类似于您的本地环境:

*   **从云存储中下载**CSV 文件
*   **在计算引擎实例上执行转换**
*   **上传**新的 CSV 文件到云存储

使用 **linux bash 脚本**，你将能够**对文件**执行如此简单的操作，比如用`[sed](https://www.gnu.org/software/sed/manual/sed.html)`替换字符，或者用`[split](https://man7.org/linux/man-pages/man1/split.1.html)`只提取你想要的列。

但是**很快**，当你不得不**构建更高级的转换**，比如合并、拆分或连接，你就有了**bash 脚本的问题和困难**。

你可以使用**更方便的脚本语言**，像 [Python 脚本和熊猫](https://pandas.pydata.org/)库，但是在内存中加载 Gb，甚至 Tb 的数据**需要时间，磁盘空间，内存使用，因此是昂贵的**。

# BigQuery 解决方案

BigQuery 是一个**强大的分析引擎，可以在几秒钟内处理 Tb 和 Pb 的数据**。当你有 CSV 文件时，有 **2 种方法用 BigQuery 查询数据**:

*   [**使用加载作业将您的数据从云存储加载到 BigQuery**](https://cloud.google.com/bigquery/docs/loading-data) 原生表中。*操作免费。*你要支付 BigQuery 上的存储*(还有不删除源文件的话云存储上的存储)。*
*   **创建一个** [**外部表**，引用 CSV 文件](https://cloud.google.com/bigquery/external-data-cloud-storage)在云存储中的位置。你将只支付云存储上的存储；**无事成 BigQuery** 。

这两种解决方案都有自己的首选用例

*   本机表提供了更好的性能和分区/集群能力。然而，在新文件或源文件更新的情况下，您必须执行新的加载和比较。
*   **外部表的查询速度较慢，但允许定义通配符文件模式** ( `my-file-*.csv`)来动态包含所有新文件，并考虑现有文件的变化。

*在当前上下文中，外部表更有趣是因为* ***我们要转换存储在云存储*** *中的 CSV 文件，但不排他。*

一个**大查询表(本地或外部)是一个抽象层**，当你编写查询时，你不必担心数据的位置。所以，**要查询数据，在**表中执行一个简单的 SQL 查询！

*在这两种情况下，你都要为你查询的数据量付费。*

那是用来加载和查询数据的。为了将数据导出到云存储，最近发布了一个全新的功能[](https://cloud.google.com/bigquery/docs/release-notes#October_14_2020)**，它在这个用例中非常有用**

> **[导出数据](https://cloud.google.com/bigquery/docs/reference/standard-sql/other-statements#export_data_statement):`EXPORT DATA`语句将查询结果导出到外部存储位置**

## **使用 BigQuery 转换 CSV 文件**

**“从外部位置读取数据”和“导出数据”这两个特性允许您**从云存储**查询数据，以及**通过**使用 BigQuery 引擎执行转换**将结果存储到云存储**。**

**为此，开始创建外部表**

```
CREATE OR REPLACE EXTERNAL TABLE mydataset.sales
OPTIONS (
  format = 'CSV',
  uris = ['gs://mybucket/sales-2019*.csv',
          'gs://mybucket/sales-2020*.csv'],
  skip_leading_rows = 1
)
```

**然后，只需查询它并将结果导出到云存储**

```
EXPORT DATA OPTIONS(
  uri='gs://mybucket/transformed/sales-*.csv',
  format='CSV',
  overwrite=true,
  header=true,
  field_delimiter=';') AS
SELECT field1, field2 FROM mydataset.sales ORDER BY field1 
```

**仅此而已！对存储在云存储中的 CSV 文件进行复杂的提取-转换-加载，只需几秒钟，只需几美元！停止写无聊的脚本来浪费你的时间！！**

# **超出**

**因为使用了 BigQuery，所以**也可以使用 BigQuery** 的所有牛逼特性。**

*   ****在转换过程中使用复杂的 SQL 和分析功能**，**
*   ****将 CSV 数据与 BigQuery 数据**连接，甚至从其他外部来源，如其他文件、 [Google Drive/Sheet](https://cloud.google.com/bigquery/external-data-drive) 、[云 SQL 数据库](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries)、……**
*   ****用 BigQuery 的** [**ML 功能**丰富输出](https://cloud.google.com/bigquery-ml/docs/introduction)**

**对于所有这些，你只需为你处理的数据量付费。没有要调配的虚拟机，没有要编写、测试和应用的脚本。只有**几个配置步骤和 SQL 语法。**
尽情享受吧！！**