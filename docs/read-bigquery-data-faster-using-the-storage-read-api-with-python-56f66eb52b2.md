# 使用 python 的存储读取 API 更快地读取 BigQuery 数据

> 原文：<https://medium.com/google-cloud/read-bigquery-data-faster-using-the-storage-read-api-with-python-56f66eb52b2?source=collection_archive---------0----------------------->

今天是一个特殊的日子，在忙碌了几周后，我回来写一篇文章，这是双重特殊，因为这是在我有幸获得谷歌云采访[:)让我们开始吧！](https://cloud.google.com/blog/products/data-analytics/google-cloud-data-hero-series-meet-antonio?fbclid=IwAR2sfomnhYHhsaLRjxvxc3ofa9UCuAmF0ydBUVfYYPIRKOGhkJjmRmbtm_4)

几周前，我在一台 [Vertex AI 笔记本](https://cloud.google.com/vertex-ai/docs/workbench/introduction)上用 python 读取一个超过 1 亿行 30 列的表格时，遇到了内存不足的问题。我发现我使用的是旧的 API，我需要使用新的 Google BigQuery [存储读取 API](https://cloud.google.com/bigquery/docs/reference/storage) 进行迁移。

# **什么是大查询存储读取** …