# 从 Netezza 迁移时，为 BigQuery 表选择分区和聚集键

> 原文：<https://medium.com/google-cloud/select-partitioning-and-clustering-keys-for-bigquery-tables-when-migrating-from-netezza-719e83b0e0e4?source=collection_archive---------0----------------------->

将数据仓库迁移到 BigQuery 时，一个关键部分是为表选择分区和集群策略。当您的表还很小的时候，在一开始就想出这个策略是很重要的，因为更改分区或集群模式需要重新构建表。

要向表中添加分区，您需要重新创建它。尽管添加聚类不需要重建表，但您仍然可以考虑这个选项。在…