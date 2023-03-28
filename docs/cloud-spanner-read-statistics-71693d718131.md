# 云扳手:读取统计数据

> 原文：<https://medium.com/google-cloud/cloud-spanner-read-statistics-71693d718131?source=collection_archive---------0----------------------->

[Cloud Spanner](https://cloud.google.com/spanner) 是 Google 完全托管的可扩展关系数据库服务。我们最近发布了一个新特性，Spanner read statistics，它允许您运行 SQL 查询来检索数据库在 1 分钟、10 分钟和 60 分钟间隔内的读取统计数据。这些读取统计数据允许您查看在您的数据库上执行的最常见和最消耗资源的读取(通过[扳手读取 API](https://cloud.google.com/spanner/docs/reads)完成)。

在这篇文章中，您将看到如何使用这些读取统计数据来识别哪些读取涉及高 CPU 使用率。

# 读取统计数据