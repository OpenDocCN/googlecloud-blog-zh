# 没有提交时间戳的云扳手更改观察器

> 原文：<https://medium.com/google-cloud/cloud-spanner-change-watcher-without-commit-timestamp-9f149adbdd7?source=collection_archive---------2----------------------->

[Google Cloud Spanner](https://cloud.google.com/spanner) 是一个全面管理、可扩展的关系数据库服务，用于区域和全球应用数据。它是第一个可扩展的、企业级的、全球分布的、高度一致的数据库服务，专为云构建，将关系数据库结构的优势与非关系水平扩展相结合。

![](img/16400067b46aef8a444b688bcf1f44a4.png)

开源的 [spanner-change-watcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-watcher) 库可以监控和发布云 spanner 数据库中数据变化的事件，作为 Java 应用程序的进程内服务。关于这个图书馆的一般介绍可以在[这里](/google-cloud/cloud-spanner-change-watcher-b77ca036459c)找到。

本文描述了如何使用 Spanner Change Watcher 来观察不包含和[提交时间戳列](https://cloud.google.com/spanner/docs/commit-timestamp#creating_and_deleting_a_commit_timestamp_column)的表*的变化。数据表中有一个提交时间戳列有利于审计，但它也有一些缺点:*

*   在 DML 语句使用[PENDING _ COMMIT _ TIMESTAMP()](https://cloud.google.com/spanner/docs/functions-and-operators#pending_commit_timestamp)函数更新提交时间戳列之后，它使得表[对于事务](https://cloud.google.com/spanner/docs/commit-timestamp#dml)的剩余部分不可读。
*   提交时间戳列不应是辅助索引的第一部分。

# 数据模型

扳手更改监视器可以通过使用 *ChangeSets* 表来监视一个或多个没有提交时间戳列的表。*变更集*表包含扳手变更观察器应该观察和报告的所有事务，实际数据表引用最后修改行的变更集。

# 创建变更集

每个包含(或可能包含)一个或多个对应该包含在变更集中的表的变更的事务，必须在*变更集*表中创建一个记录。这可以手动完成，或者通过使用扳手变化监视器库提供的[数据库客户端包装器](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/DatabaseClientWithChangeSets.java)来完成。后者将自动为每个读/写事务创建一个更改集记录。

## 手动创建更改集

您的应用程序可以为每个事务手动创建一个更改集记录，方法是为更改集生成一个唯一的 id，并向事务添加一个变异。同一事务中的所有变更或更新都应该引用唯一的变更集 id。

## 自动创建变更集

变更集记录也可以通过使用包装的数据库客户机自动生成。这将减少变更集应用程序中锅炉板代码的数量。应用程序仍然必须为它执行的每个变异和更新添加对变更集的引用。

# 创建扳手观察器

一旦您有了一个包含*变更集*表的数据模型，并且您的数据表包含对变更集的引用，您就可以创建一个扳手变更监视器，它将拾取并报告底层数据表的所有变更。

扳手变化观察器库包含观察器的构建器，可以观察完整的数据库([SpannerDatabaseChangeSetPoller.java](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerDatabaseChangeSetPoller.java))以及单个表格([SpannerTableChangeSetPoller.java](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerTableChangeSetPoller.java))。

# 将更改发布到发布订阅

扳手变化观察器库还包含[扳手变化发布者](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-publisher)库。这个库可以用来发布由扳手变化监视器向 PubSub 报告的变化。扳手变更发布者接受[SpannerDatabaseChangeWatcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerDatabaseChangeWatcher.java)和[SpannerTableChangeWatcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerTableChangeWatcher.java)接口的任何实现。这意味着变更集轮询器也可以用于将变更发布到 PubSub。

# 更多样本

请参见[https://github . com/cloudspannerlecosystem/Spanner-change-Watcher/tree/master/samples](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/samples)了解有关如何使用云扳手观察器和云扳手发布器的更多示例，包括以下示例:

*   错误处理
*   配置轮询间隔
*   使用自定义提交时间戳存储库
*   使用[分片](https://cloud.google.com/spanner/docs/whitepapers/optimizing-schema-design#anti-pattern_timestamp_ordering)允许对提交时间戳列进行索引，以优化轮询