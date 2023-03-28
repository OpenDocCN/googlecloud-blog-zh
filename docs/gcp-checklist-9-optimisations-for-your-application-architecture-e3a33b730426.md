# GCP 清单 9 —优化您的应用架构

> 原文：<https://medium.com/google-cloud/gcp-checklist-9-optimisations-for-your-application-architecture-e3a33b730426?source=collection_archive---------1----------------------->

有一些常见的应用程序模式，如 3 层 web 服务或事件驱动管道。GCP 为您提供构建模块和托管服务来满足您的需求。构建应用程序的基础取决于许多因素。什么类型的[计算](https://cloud.google.com/docs/choosing-a-compute-option)和什么类型的[存储](https://cloud.google.com/storage-options/)是要问的典型问题。下面的参考资料部分提供了一些链接，可以帮助您回答一些基础架构问题。

从[解决方案页面](http://cloud.google.com/solutions)中选择一个基础架构，您就可以快速起步。你将需要挖一点周围，但有一些宝石在那里我保证！

一旦您知道您的应用程序将基于什么体系结构，您将需要调整本指南中提供的通用指南，以反映安全考虑、访问、监控和警报要求。

*   安全性考虑—应用程序级别的安全性以及所使用的服务将具有您需要关注的特定控制
*   访问权限—用户如何访问您的应用程序。你如何保护特定于你的应用的 API？根据指南保护您的 GCP 环境
*   选择适合您使用情形的计算、存储和网络产品及功能
*   可靠性设计
*   延迟、I/O、数据库设计都会对您的应用程序性能产生影响衡量并了解您的计算选择、网络配置和数据库设计的影响

你的阅读清单(你知道它要来了！) :

[https://cloud.google.com/terms/services](https://cloud.google.com/terms/services)

[选择一个计算选项](https://cloud.google.com/docs/choosing-a-compute-option)

[https://cloud.google.com/compute/docs/disks/performance](https://cloud.google.com/compute/docs/disks/performance)

[确定 GKE 集群的规模和范围](https://cloud.google.com/solutions/scope-and-size-kubernetes-engine-clusters)

[选择一个存储选项](https://cloud.google.com/storage-options/)

[将大数据集转移到 GCP](https://cloud.google.com/solutions/transferring-big-data-sets-to-gcp)

[选择连接选项](https://cloud.google.com/interconnect/)

[网络层](https://cloud.google.com/network-tiers/)

【https://firebase.google.com/docs/auth/ 

[解决方案架构网站](http://gcp.solutions/)

[教程和指南](https://cloud.google.com/docs/tutorials)

[http://www.gcping.com/](http://www.gcping.com/)

[https://cloudprober.org](https://cloudprober.org)

你的清单是:

可以在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到该系列中所有清单的列表