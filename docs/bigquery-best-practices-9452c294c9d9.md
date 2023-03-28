# BigQuery 最佳实践

> 原文：<https://medium.com/google-cloud/bigquery-best-practices-9452c294c9d9?source=collection_archive---------0----------------------->

## [数据工程 101](https://towardsdatascience.com/tagged/data-engineering-101)

## BigQuery 中查询性能优化和工作流自动化的实用技巧。

![](img/083302f24cbbeaf0cf356b1cc095a802.png)

由[马克-奥利维尔·乔多因](https://unsplash.com/@marcojodoin?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)在 [Unsplash](https://unsplash.com/s/photos/speed?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上拍摄的照片

BigQuery 是 GCP 上提供的一个完全托管和高度可伸缩的数据仓库。我的团队采用 BigQuery 作为所有数据分析用例的集中式数据仓库，以支持数据驱动的决策制定。在这个…