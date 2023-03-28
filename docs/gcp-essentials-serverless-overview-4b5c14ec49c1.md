# GCP Essentials—无服务器概述

> 原文：<https://medium.com/google-cloud/gcp-essentials-serverless-overview-4b5c14ec49c1?source=collection_archive---------2----------------------->

《GCP 要点》的新一集现已推出，这一集是对之前的[GCP 计算平台概述](/google-cloud/new-gcp-essentials-episode-gcp-compute-platform-overview-cc73d583d9d0)的后续，重点介绍 GCP ***无服务器计算*** 选项:

当然，这个短视频涵盖了作为无服务器范式的流行实现的谷歌云功能，以及新发布的云运行和令人尊敬的应用引擎。

[**云函数**](http://cloud.google.com/functions) 提供了一个事件驱动的平台，少量代码对事件或 HTTP 请求做出反应。支持多种语言和丰富的库。环境变量、安全设置和简化的认证使云功能成为利用谷歌机器学习、存储或大数据 API 的最简单方式之一。

[**Cloud Run**](http://cloud.run) 让您以无服务器的方式运行您的无状态容器(扩展到零，多并发，以及一个伟大的[定价模型](https://cloud.google.com/run/pricing)),其额外的好处是构建在 Knative 上，因此能够在 GKE 上的 Cloud Run 上部署完全相同的工作负载。部署容器显然给了开发人员在语言和框架方面完全的自由。

[**App Engine**](http://cloud.google.com/appengine) 对于基于微服务的应用来说仍然是一个非常合适的选择。它提供了带有内置版本控制的最新流行语言运行时，使得快速独立地扩展和迭代服务变得容易。流量分流也使得 A/B 测试和分阶段部署变得非常简单。

该视频还简要介绍了其他工具和产品，它们将帮助您构建跨越云功能、云运行和/或应用引擎的应用。即云发布/订阅、云调度程序和云任务。

这实际上归结为您希望移交给运行时的工件:一个函数、一个容器或一个应用程序。GCP 可以处理所有这些，并以无服务器的方式大规模运行它们:无需设置或管理集群/基础架构/安全性，自动扩展(包括到零)，以及按使用付费。

你也应该看看这个伟大的[云运行常见问题](https://github.com/ahmetb/cloud-run-faq)。

请继续关注即将播出的 [GCP 精华集](https://goo.gle/2w9xfHR)并订阅 [GCP YouTube 频道](http://youtube.com/googlecloudplatform)。