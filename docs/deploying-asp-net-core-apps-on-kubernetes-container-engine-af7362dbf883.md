# 在 Kubernetes/Container 引擎上部署 ASP.NET 核心应用

> 原文：<https://medium.com/google-cloud/deploying-asp-net-core-apps-on-kubernetes-container-engine-af7362dbf883?source=collection_archive---------0----------------------->

在我之前的[帖子](https://meteatamel.wordpress.com/2017/08/15/deploying-asp-net-core-apps-on-app-engine/)中，我谈到了如何在 Google Cloud 上部署一个容器化的 ASP.NET 核心应用到应用引擎(flex)。App Engine (flex)是一种在生产中运行容器的简单方法:只需发送您的容器，让 Google Cloud 计算出如何大规模运行它。它有一些不错的默认功能，如版本控制、流量分流、仪表盘和自动缩放。然而，它并没有给你太多的控制权。

有时，您需要创建一个容器集群，并控制每个容器的部署和伸缩方式。这时候 [Kubernetes](https://kubernetes.io/) 开始发挥作用。Kubernetes 是一个开源的容器管理平台，可以帮助你管理一个容器集群，容器引擎是由 Google Cloud 管理的 Kubernetes。

在这个“云一分钟”中，我展示了如何将一个 ASP.NET 核心应用程序部署到运行在容器引擎上的 Kubernetes。

如果你想自己经历这些步骤，我们也有一个代码实验室，你可以在这里访问[。](https://codelabs.developers.google.com/codelabs/cloud-kubernetes-aspnetcore)

*原载于 2017 年 9 月 11 日*[*meteatamel.wordpress.com*](https://meteatamel.wordpress.com/2017/09/11/deploying-asp-net-core-apps-on-kubernetescontainer-engine/)*。*