# 没有互联网连接的完全私有的 GKE 集群

> 原文：<https://medium.com/google-cloud/completely-private-gke-clusters-with-no-internet-connectivity-945fffae1ccd?source=collection_archive---------0----------------------->

![](img/a01c9cda72e9da2556294ab4b4827028.png)

将您的 Google Kubernetes 引擎(GKE)集群与互联网访问隔离有几个原因，其中最主要的原因是安全性。对于许多金融、政府和类似机构来说，这是在谷歌云平台(GCP)上运行其工作负载的必要条件。

这篇文章将介绍在网络中建立一个孤立的 GKE 集群所需的步骤…