# 云的隐性成本

> 原文：<https://medium.com/google-cloud/the-hidden-costs-of-cloud-ddb702495e93?source=collection_archive---------0----------------------->

在[服务器密度](https://www.serverdensity.com/)时，我们刚刚完成了一个为期 9 个月的项目，将我们所有的工作负载从 Softlayer 迁移到 Google 云平台。[这始于使用云 Bigtable](https://blog.serverdensity.com/time-series-data-opentsdb-bigtable/) 的单一服务，用于大规模存储时间序列监控数据，并以我们现在在 GCP 运行的整个产品而告终。

迁移进行得很顺利，我们很高兴能使用谷歌云。虽然与市场领导者(AWS)相比，他们的产品较少，但我们认为谷歌云产品设计更好，定价模型更容易使用，并且定期发布创新的新功能…