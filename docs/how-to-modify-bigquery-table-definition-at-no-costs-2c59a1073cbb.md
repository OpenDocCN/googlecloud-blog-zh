# 如何免费修改 Bigquery 表

> 原文：<https://medium.com/google-cloud/how-to-modify-bigquery-table-definition-at-no-costs-2c59a1073cbb?source=collection_archive---------0----------------------->

你在用 BigQuery 吗？您是否需要对表进行更改，比如更改分区键、键入或重命名列？阅读这个技巧可以节省运行 BigQuery 作业的成本。

![](img/a8d5343a8a55be8d48e304ac5cfaf581.png)

在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上由 [micheile dot com](https://unsplash.com/@micheile?utm_source=medium&utm_medium=referral) 拍摄的照片

BigQuery 是 Google 大数据分析工具的旗舰。没有免费的东西，强大的功能会带来成本。在大查询的情况下，成本有两种类型:每次存储和每次执行分析…