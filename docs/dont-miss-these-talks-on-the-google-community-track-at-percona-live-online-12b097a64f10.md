# 不要错过在 Percona Live Online 的 Google 社区轨道上的这些演讲

> 原文：<https://medium.com/google-cloud/dont-miss-these-talks-on-the-google-community-track-at-percona-live-online-12b097a64f10?source=collection_archive---------1----------------------->

![](img/a4bcc40fb5270cb5c98b4d85d247ab54.png)

2021 年 5 月 12 日至 13 日，请加入我们在 [Percona 在线直播](https://events.percona.com/events/details/percona-virtual-presents-percona-live-online/)的[谷歌社区跟踪](https://perconaliveonline.sched.com/overview/type/Community+Track/Google)，参加从使用 Kubernetes 数据库到数据库迁移、可观察性和故障排除等话题的讨论。Percona Live 是一个面向社区的活动，面向数据库开发人员、管理员和决策者，与同行和技术专业人员进行交流。

这是我们将在活动中展示的一些演讲的预览。

**如何选择合适的解决方案将您的数据库迁移到云**

产品经理 Shachar Guz

数据库迁移可能复杂、耗时且成本高昂，增加了迁移到云的风险。企业需要有效可靠的数据库迁移工具来帮助自动化这一过程，以实现无缝的云采用。

在本次会议中，我们将详细阐述选择数据库迁移方法和技术时要考虑的因素，以及可能影响风险的不同方面，如迁移保真度和停机时间。您将了解如何考虑迁移过程中的关键方面，如准备、连接和停机时间，以及如何利用[数据库迁移服务](https://cloud.google.com/database-migration?utm_source=blog&utm_medium=partner&utm_campaign=CDR_jkl_databases_perconalive_)实现迁移成功。

**使用 GKE 部署高可用性 PostgreSQL】**

[Christoph Bussler](https://twitter.com/chbussler) ，解决方案架构师和 Shashank Agarwal，数据库迁移工程师

当您在 [Google Kubernetes 引擎](https://cloud.google.com/kubernetes-engine?utm_source=blog&utm_medium=partner&utm_campaign=CDR_jkl_databases_perconalive_)中运行一个应用程序时，对于如何部署数据库有多种选择和考虑。在本专题讲座中，您将了解在 GKE 选择数据库部署选项时的一些体系结构考虑因素。我们将演示其中的一个选项，您将了解如何基于区域持久磁盘和 PersistentVolumeClaims 将 PostgreSQL 配置为 GKE 的容器。在区域永久磁盘上运行 PostgreSQL 可在区域停机时提供零 RPO，我们将演示如何进行故障转移。

**使用 SQL Commenter 和云 SQL Insights 揭开数据库性能问题的神秘面纱**

[开发商代言人 Jan Kleinert](https://twitter.com/jankleinert)

您是否尝试过对使用 ORM 构建的应用程序中的数据库性能问题进行故障诊断？ORM 可以简化与数据库通信的应用程序的开发，但是由于 ORM 正在生成 SQL 语句，所以很难确定哪个应用程序代码导致了缓慢的查询。

[SQL Commenter](https://google.github.io/sqlcommenter/) 是一个开源库，它使 ORM 能够用关于导致其执行的代码的注释来增加 SQL 语句，从而更容易将您的应用程序代码与 ORM 生成的 SQL 语句相关联。

在本节中，我们将演示如何设置和使用 SQL Commenter 以及一个使用 Knex 来诊断查询性能的应用程序。我们还将涉及 SQL Commenter 支持的其他框架和表单，以及如何在数据库日志和观察工具中查看这些数据，包括 [Cloud SQL Insights](https://cloud.google.com/sql/docs/postgres/insights-overview?utm_source=blog&utm_medium=partner&utm_campaign=CDR_jkl_databases_perconalive_) 。

**在 Kubernetes 上有效使用云 SQL**

[Eno Compton](https://twitter.com/enocom_) ，开发者关系工程师

Kubernetes 和 [Cloud SQL](https://cloud.google.com/sql?utm_source=blog&utm_medium=partner&utm_campaign=CDR_jkl_databases_perconalive_) 有很多令人喜欢的地方，但是如何一起使用它们并不总是很清楚。本次演讲将深入探讨云 SQL 以及如何在 Kubernetes 上有效运行它。演讲将以开发人员为中心，从从 Kubernetes 连接到云 SQL 的基础知识开始。它将包括如何运行负载测试，然后确定实例的大小。最后，它将介绍如何启用连接池来充分利用云 SQL，同时享受其中的乐趣。

我们希望您能参加我们的会议和问答，更详细地了解这些主题和技术。活动期间，演讲者将出现在问答聊天中。[谷歌社区跟踪](https://perconaliveonline.sched.com/overview/type/Community+Track/Google)会议将于 5 月 13 日举行，但请务必查看[的全部日程](https://perconaliveonline.sched.com/)并在今天注册[！](https://events.percona.com/events/details/percona-virtual-presents-percona-live-online/)