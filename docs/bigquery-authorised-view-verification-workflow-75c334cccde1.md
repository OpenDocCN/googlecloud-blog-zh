# BigQuery 授权视图验证工作流

> 原文：<https://medium.com/google-cloud/bigquery-authorised-view-verification-workflow-75c334cccde1?source=collection_archive---------0----------------------->

![](img/1f3064b5874cdb5434e132067f1229f0.png)

保罗·斯科鲁普斯卡斯在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上的照片

***TL；DR:在 BigQuery 数据集中验证您的视图，以确保授权的视图能够工作而不会中断您的 ETL。***

客户希望验证 BigQuery 数据仓库(DW)上的所有视图，以评估是否所有视图都根据 DW 内部或外部的现有数据集进行了授权。

BigQuery API 还没有为此实现一个方法，所以我们选择使用 BigQuery 对象开发一个 Python 解决方案，收集项目、数据集、表级别的信息。

## ***用例 1(添加到 DW 中的新数据集):***

> 一个新的数据集被添加到 DW 中。
> 
> 启动一个流程，收集所需的信息。
> 
> 查看数据集，寻找视图。
> 
> 查看每个视图，并收集 SQL 代码。
> 
> 从 SQL 获取外部数据源。
> 
> 获取现任数据集，并验证该视图已针对该数据集获得授权。
> 
> 否则，向源数据集和目标数据集的所有者发送通知(电子邮件、空闲时间等)，以便他们采取措施。
> 
> 数据集被验证为授权的，并被添加到目录系统中。

## **用例 2(定期检查现有数据集，寻找变化):**

> 用例 1 的重复验证。
> 
> 每隔(天、周……)对数据仓库上的所有数据集和视图启动一次批量验证。
> 
> 创建一个报告，抛出异常，并向数据集所有者发送单独的通知。

## ***解决方案:***

**步骤 1 —在给定项目上列出所有数据集。**这一步将收集给定项目的所有数据集。 *Project = client.project* (包含当前项目)。

运行该代码，输出将如下所示:

```
Datasets in project bq-sme:dataset1dataset2
```

参考资料:

[https://cloud . Google . com/big query/docs/listing-datasets # python](https://cloud.google.com/bigquery/docs/listing-datasets#python)

[https://cloud . Google . com/big query/docs/reference/rest/v2/datasets/list](https://cloud.google.com/bigquery/docs/reference/rest/v2/datasets/list)

收集数据集列表将使我们进入下一步。

**步骤 2——在给定的数据集上，列出所有视图。**这一步将收集给定数据集的所有视图。

运行该代码，输出将如下所示:

```
Got dataset 'bq-sme.dataset1' with friendly_name 'None'.Description: sample dataset 1Labels:type: sampleTables:Table1 - TABLEView1 - VIEWView2 - VIEW
```

参考资料:

[https://cloud . Google . com/big query/docs/dataset-metadata # python](https://cloud.google.com/bigquery/docs/dataset-metadata#python)

[https://cloud . Google . com/big query/docs/reference/rest/v2/tables/list](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/list)

该示例代码将检索所有的表，因此您只能获得该示例的类型视图。

**步骤 3——获取视图元数据，收集视图的 SQL 定义。**这一步将收集给定视图的元数据，检索相应的 SQL 元数据。

元数据将告诉我们在这个视图中是否使用了外部数据集。

运行该代码，输出将如下所示:

```
View at bq-sme:dataset1.View1View Query:SELECT station_id FROM `bq-sme.dataset2.ny_citibike`
```

参考资料:

[https://cloud.google.com/bigquery/docs/view-metadata#python](https://cloud.google.com/bigquery/docs/view-metadata#python)

[https://cloud . Google . com/big query/docs/reference/rest/v2/tables/get](https://cloud.google.com/bigquery/docs/reference/rest/v2/tables/get)

正如我们所看到的，数据集上有一个视图(表类型视图)，指向另一个数据集，因此这是一个需要授权的视图的示例。

**步骤 4——验证一个数据集是否包含另一个数据集中视图的授权。**获取数据集元数据，查看访问属性并验证给定视图是否在授权视图列表中。

运行该代码，输出将如下所示:

```
Got dataset 'bq-sme.dataset2' with friendly_name 'None'.Description: sample dataset 2Labels:type: sampleAccess:[<AccessEntry: role=WRITER, specialGroup=projectWriters>, <AccessEntry: role=OWNER, specialGroup=projectOwners>, <AccessEntry: role=OWNER, userByEmail=email@domain.com>, <AccessEntry: role=READER, specialGroup=projectReaders>, <AccessEntry: role=None, view={u'projectId': u'bq-sme', u'tableId': u'View1', u'datasetId': u'dataset1'}>]
```

参考资料:

[https://cloud . Google . com/big query/docs/dataset-metadata # python](https://cloud.google.com/bigquery/docs/dataset-metadata#python)

[https://cloud . Google . com/big query/docs/reference/rest/v2/datasets/get](https://cloud.google.com/bigquery/docs/reference/rest/v2/datasets/get)

[https://cloud . Google . com/big query/docs/reference/rest/v2/datasets # Dataset](https://cloud.google.com/bigquery/docs/reference/rest/v2/datasets#Dataset)

有了这个响应，您就可以从原始数据集获取一个视图，并结束这个循环。

如果没有找到授权，您可以向数据集所有者发送通知，以便解决问题。

## 可扩展性:

*   将代码部署在作为谷歌应用引擎(GAE)的可扩展任务管理器上，将对给定的数据集或视图进行原子请求，并且可以管理请求的吞吐量，并管理潜在的 100 或 1000 个数据集和视图。
*   此外，云功能或 GKE 或云运行，可以以自动化的方式为 BigQuery 上的任何新视图执行此过程，并为当天验证运行夜间运行。

## 进化:

本文中描述的过程可以扩展到 Bigquery 上的表 ACL，并且有可能验证给定的表是否与现任组或用户共享。