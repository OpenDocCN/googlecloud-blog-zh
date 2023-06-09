# 如何备份 Google BigQuery

> 原文：<https://medium.com/google-cloud/how-to-backup-google-big-query-5f078138cedc?source=collection_archive---------0----------------------->

![](img/6ca94db6a60d1ecf06eb5aea65ab7a1e.png)

# 假设

备份策略基于以下假设:

*   BigQuery 中的数据大小:50TB
*   数据摄取方法:仅追加(将数据加载到新的每日分区中，保留旧分区中的所有历史数据)
*   加载频率:每天
*   BQ 中的当前数据保留期:数据存储 2 年，没有任何删除或存档
*   每日增量负载大小:50/(2 * 365)～= 70GB
*   每 3 年可能需要一次完整的备份恢复(例如，整个数据集或项目被意外删除，但很晚才被发现)
*   一个季度可能需要一次部分(分区或表)备份恢复(例如:错误的 ETL 操作截断的表或分区)
*   所有 1 年的数据都应保留
*   1 年后，必要的摘要/汇总数据可以单独备份(大小可以忽略不计),每日备份可以删除。

# 备份策略

BQ 在后台执行自动快照，可在 2 天内恢复(对于丢弃的数据),在 7 天内恢复表数据。以下几点和备份策略可用于从恢复操作中恢复，对于表数据，这些操作在 2 天内不会被注意到，对于 7 天内不会被注意到。有关快照和取消删除的更多信息，请参考[本](https://cloud.google.com/bigquery/table-decorators#snapshot_decorators)和[本](https://cloud.google.com/bigquery/docs/managing-tables#undeletetable)。

**备选方案 1**(GCS 中的成本效益选项):

*   将所有当前数据一次性备份到 GCS
*   ETL 窗口结束后，每天备份增量负载(新分区和新表)
*   使用 GCS 单区域
*   将 coldline 存储用于当前备份和超过 1 个月的每日增量备份
*   使用近线存储进行 30 天的每日增量备份
*   30 天后，将每日增量备份从近线移动到冷线
*   1 年后，从 coldline 中删除每日增量备份
*   在另一个 GCP 项目中使用 bucket 也可以从意外的项目删除中恢复，并提供单独的 IAM 特权和访问范围。(注意:项目所有者可以在 30 天内停止删除。这种情况是最悲观的，但不会增加任何额外成本。)
*   采用这种策略，GCS 中会有一个副本

**备选项 2** (另一个数据集和项目中的 1 个历史副本) :

*   将所有当前数据集一次性复制到专门为备份创建的项目中名称中包含备份日期的其他数据集。
*   每天复制新分区和新表，如果其他数据没有变化，则追加到备份数据集。
*   在这个示例策略中，将有一个保存所有历史的备份副本

**备选 3** (多余的副本；如果数据变化是不可预测的，例如，未控制数量的表被截断和加载，先前分区中的数据被更新等):

*   每天将所有当前数据集复制到专门为备份创建的项目中名称中包含备份日期的其他数据集
*   定义数据集删除策略，例如:
*   将每周开始的备份数据集保留 4 周，并在 1 周后删除其他每日备份数据集。
*   将月初的备份数据集保留 3 个月，并在 1 个月后删除其他每周备份数据集。
*   使用这个示例策略，BigQuery 中将有 7+3+2=12 个副本
*   根据业务需求，可以缩小尺寸。

# 费用

*   存储在 coldline 中的总备份大小:50TB+70GB * 335 ~ = 73.5 TB
*   近线存储的总备份大小:70GB*30=2.1TB
*   单个地区的 Coldline 存储成本为 73,5TB(例如:芬兰):每月 301.06 美元
*   单个地区(例如芬兰)2.1TB 的近线存储成本:每月 21.50 美元
*   **总费用:每月 322.56 美元**

注意价格可能会有变化，请查看 [GCS 价格页面](https://cloud.google.com/storage/pricing)了解价格详情。

您还可以使用 [GCP 价格计算器](https://cloud.google.com/products/calculator)计算最新价格的成本。

# 75TB 的其他存储替代方案

*   Coldline 单地区:每月 307.20 美元
*   多地区热线:每月 537.60 美元
*   近线多地区:每月 768 美元

对于其他计算:[https://cloud.google.com/products/calculator/](https://cloud.google.com/products/calculator/)

对于存储定价:[https://cloud.google.com/storage/pricing](https://cloud.google.com/storage/pricing)