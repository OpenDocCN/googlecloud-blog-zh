# 无服务器:谷歌云无运营数据架构之旅

> 原文：<https://medium.com/google-cloud/serverless-a-journey-to-a-no-ops-data-architecture-on-google-cloud-a3b019a27799?source=collection_archive---------3----------------------->

在数据世界中，无服务器确实是一个新奇的术语。承诺最大限度地减少对基础设施的担忧( [no-ops](https://www2.deloitte.com/us/en/insights/focus/tech-trends/2019/noops-serverless-computing-transforming-it-operations.html) )并只为您使用的东西付费似乎是一个梦想，这促使我进一步回顾 GCP 所有可用的无服务器产品，然后为一个简单的项目构建一个涵盖数据编排、数据处理和数据仓库基础知识的架构。

# **用例**

为了保持这种架构的可复制性，我将使用来自谷歌的免费 [COVID 数据集](https://cloud.google.com/blog/products/data-analytics/free-public-datasets-for-covid19)…