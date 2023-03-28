# Google Analytics 数据传输到 BigQuery

> 原文：<https://medium.com/google-cloud/google-analytics-data-transfer-to-bigquery-fad388ae646a?source=collection_archive---------4----------------------->

![](img/6bf60aa3bde35a6603505fc26480bd06.png)

**简介**

**在 BigQuery 中获取网站访客信息。**

在本文中，我们将了解如何将实时数据从 google analytics 转移到 BigQuery。使用谷歌分析，我们可以跟踪访问我们网站的访问者的数据，访问者浏览器，访问者位置等。

**步骤 1** 创建样本网站。

1.  转到[https://sites.google.com](https://sites.google.com/new)

![](img/f3ed8b85eb1bc461f64c2c4ffc894ead.png)

2.创建示例网站。

![](img/8e3ddbccecad7ae0e9e85aa0f49ef866.png)

3.添加自定义域名到您的谷歌网站(可选)

设置>自定义域

![](img/b7d57ab645267c3285b4b01c6204b606.png)

**步骤 2——创建谷歌分析账户**

**第 3 步——在谷歌网站中添加谷歌分析跟踪 ID，以跟踪我们网站的流量。**

![](img/b9806d59e1f188bf16b79558bf352cdb.png)

**步骤 4 验证显示数据的谷歌分析仪表盘**

![](img/05cacf3324cd40e1e701ca5eb2df78fa.png)

步骤 5 将数据传输到 Bigquery

1.  启用 **BigQuery 数据传输** API
2.  导航到谷歌分析的管理标签

![](img/e2329c1659f61f7e54f175ef5bd9c719.png)![](img/8f78645550dd333a85dcd7c8a8d2ed81.png)![](img/08197077a53ff89cb7f52073e0552785.png)

2.点击**大查询链接**

![](img/08197077a53ff89cb7f52073e0552785.png)

3.使用项目 Id 搜索并链接 Google Cloud Bigquery 项目。

![](img/c8664bb89d3f9c002fb40273d41e7ca9.png)

4.将频率配置为实时数据流。

![](img/c381b810eefb946a7b422c3f89483671.png)![](img/cbcdaaa5ec05539e0fa059d53bb2354f.png)

现在，所有与网站访问者和流量相关的数据都将开始加载到 BigQuery 中。

![](img/ef811a3b6d86a907ae39718fbe45dd1c.png)

通过在 BigQuery 中运行查询来验证数据。

![](img/5f76640f96d52dd218c62e945e6764a2.png)