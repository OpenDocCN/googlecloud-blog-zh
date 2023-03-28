# 云扳手变化观察器

> 原文：<https://medium.com/google-cloud/cloud-spanner-change-watcher-b77ca036459c?source=collection_archive---------0----------------------->

[Google Cloud Spanner](https://cloud.google.com/spanner) 是一个全面管理、可扩展的关系数据库服务，用于区域和全球应用数据。它是第一个可扩展的、企业级的、全球分布的、高度一致的数据库服务，专为云构建，将关系数据库结构的优势与非关系水平扩展相结合。

![](img/16400067b46aef8a444b688bcf1f44a4.png)

谷歌云扳手

多个应用程序和服务可以同时与同一个 Cloud Spanner 数据库交互，常见的模式是一个服务基于某些事件触发其他服务，比如在数据库中插入或更新记录。本文描述了开源的 [spanner-change-watcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-watcher) 库，它可以作为 Java 应用程序的进程内服务来监控和发布云 spanner 数据库中的数据更改事件。

开源库还包含了 [spanner-change-publisher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-publisher) 服务，它可以作为一个独立的应用程序运行，将从云 spanner 数据库发布对一个或多个 Pubsub 主题的更改。关于这项服务的介绍可以在[这里](/@knutolavloite/cloud-spanner-change-publisher-7fbee48f66f8)找到。

# 先决条件和限制

*   扳手变化监视器使用[提交时间戳](https://cloud.google.com/spanner/docs/commit-timestamp)来确定行何时被更新。使用这个库只能监视包含具有上次更新提交时间戳的列的表。这个专栏的名字可以自由选择。
*   该库只能检测插入和更新，因为它依赖行的提交时间戳值来检测更改。要检测删除，您应该通过在行上设置 deleted 标志来实现逻辑删除，而不是实际删除它。如果在带有一个或多个标记为 ON DELETE CASCADE 的子表的父表上执行删除，那么子表也需要相应地更新，以便由观察器选取。单独的后台任务可以使用该标志来检测稍后应该删除哪些行。
*   该库通过轮询被监视的表来监视数据库的变化。轮询间隔默认为一秒钟，可以进行配置。如果在小于轮询间隔的时间范围内对同一行有多次更改，库可能只报告最后一次更改。

# 成分

扳手更换观察器项目包含三个主要组件:

*   Spanner Change Watcher:一个 Java 后台服务，将监视云 Spanner 数据库的一个、一些或所有表的变化，并将这些变化作为进程内事件发布到应用程序。这个库可以集成到需要通知数据更新的现有 Java 应用程序中。本文将重点讨论这个组件。
*   Spanner Change Publisher:一个独立的应用程序，它将观察云 Spanner 数据库的一个、一些或所有表的变化，并将这些变化以 JSON 或 Avro 格式发布到 Pubsub。可以将应用程序配置为将所有更改发布到一个主题，或者可以将更改发布到每个表的单独主题。参见[这篇文章](/@knutolavloite/cloud-spanner-change-publisher-7fbee48f66f8)了解扳手更换发布器组件的更多信息。
*   Spanner Change Archiver:一个示例应用程序，使用 Spanner Change Publisher 将更改事件发布到 Pubsub 主题，这将再次触发 Google Cloud 函数，将更改写入 Google 云存储。

# 扳手更换监视器

扳手变化观察器是一个进程内后台服务，可以包含在现有的 Java 应用程序中。它将观察一组云扳手表的变化，并通过回调通知应用程序检测到的任何变化。

下面是一个简单的回调示例，它将更新行的内容打印到控制台。

扳手更换观察器—回拨样本

Spanner Change Watcher 允许您为单个表、多组表或整个数据库创建观察器。可以为每个观察器分配一个或多个回调，这些回调将在检测到更改时接收通知，包括新数据和更改的提交时间戳。

对一个表的更改保证按照提交时间戳的顺序传递给回调，并且库保证在任何给定的时间，对于一个表，最多有一个回调是活动的。当监视多个表时，这些表被并行轮询，并且不能保证对不同表的回调的顺序。这意味着对`TABLE1`的改变的回调可能在对`TABLE2`的回调之前被调用，即使对`TABLE2`的改变发生在对`TABLE1`的改变之前。

## 看着一张桌子

单个表的观察器实现接口[SpannerTableChangeWatcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerTableChangeWatcher.java)。目前，这个接口的唯一实现是 [SpannerTableTailer](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerTableTailer.java) ，它以固定的时间间隔轮询被监控的表。

创建一个 SpannerTableTailer

## 观察整个数据库

整个数据库或数据库中一组表的观察器实现了接口[SpannerDatabaseChangeWatcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerDatabaseChangeWatcher.java)。目前，这个接口的唯一实现是[spannerdatabasetiler](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerDatabaseTailer.java)，它以固定的时间间隔轮询每个被监控的表。这些表相互独立地被并行轮询。

创建 SpannerDatabaseTailer

在创建`SpannerDatabaseTailer`时使用选项`allTables()`将自动创建一个观察器，该观察器将观察所有具有提交时间戳列的表*。没有 commit timestamp 列的所有表都将被忽略，并且不需要从要监视的表集中显式排除。*

也可以将某些表排除在监视范围之外。使用选项`allTables().except(String...)`将特定的表排除在监视范围之外。

# 教程:示例应用程序

现在，我们将通过一步一步的指南来创建一个简单的应用程序，该应用程序将观察两个表的更改并将所有更改写入控制台。完整的[示例应用程序可以在这里找到](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/samples/spanner-change-watcher-samples/src/main/java/com/google/cloud/spanner/watcher/sample/SimpleChangeWatcherSample.java)。

## 数据模型

此示例假设数据库中存在以下表。如果它们还不存在，提供的示例应用程序将自动为您创建它们。

每个表都包含一个时间戳类型的 LastUpdateTime 列，选项为 allow_commit_timestamp=true。这意味着 Cloud Spanner 可以使用更新行的事务的提交时间戳自动设置列。扳手变化监视器只能监视具有这种列的表。可以自由选择列名，库将自动检测使用哪一列。

样本数据模型

## 属国

扳手变化监视器是你的项目中需要的一个依赖项。

扳手变化观察者相关性

## 创建扳手更换观察者

随着 Google-cloud-spanner-change-watcher 添加到您的项目中，您现在可以创建一个变更观察器了。对于这个例子，我们将为整个数据库创建一个观察器。该观察器将自动包括数据库中所有具有提交时间戳列的表。所有没有提交时间戳列的表都将被忽略。

您还可以设置一组特定的表来监视，或者设置监视程序来监视除了一组特定的表之外的所有表。参见 [SpannerDatabaseTailer 的文档。Builder](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerDatabaseTailer.java) 了解更多信息。

为所有表创建数据库监视器

您可以向观察器添加一个或多个[回调](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/8948e6df61dc8d39474bbdf34d285d3c8e5b229e/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/SpannerTableChangeWatcher.java#L37)，以接收检测到的所有更改的通知。本例中的回调只是打印已经更新到控制台的行的新内容。

## 写入更改

写入扳手数据库的所有更改现在也将由观察器打印到控制台。示例应用程序将 5 位歌手和 5 张专辑写入数据库。重要的是，向 Cloud Spanner 写入数据的应用程序也用当前事务的提交时间戳填充提交时间戳列。使用变异对象时使用`Value.COMMIT_TIMESTAMP`占位符值，通过 DML 写入数据时使用`PENDING_COMMIT_TIMESTAMP()`函数。

使用价值。COMMIT_TIMESTAMP 在使用变异对象时写入提交时间戳

## 运行示例应用程序

要执行示例应用程序，请按照下列步骤操作:

1.  [从 GitHub](https://github.com/cloudspannerecosystem/spanner-change-watcher) 克隆项目。
2.  导航到`spanner-change-watcher/samples/spanner-change-watcher-samples`文件夹。
3.  通过执行`mvn clean package`创建一个. jar。
4.  执行以下命令，其中`<instance-id>`和`<database-id>`必须用实际值代替。该实例必须已经存在，如果数据库尚不存在，将自动创建该数据库。

```
java -cp target/spanner-change-watcher-samples.jar \
com.google.cloud.spanner.watcher.sample.SimpleChangeWatcherSample \
<instance-id> <database-id>
```

示例应用程序将启动，控制台的输出应该如下所示。

抽样输出

如果您通过任何其他应用程序在数据库中插入或更新更多数据，例如 Google Cloud Spanner web 界面或 gcloud util，观察器也会打印出这些更改。

# 更多样本

关于如何使用 spanner-change-watcher 的更多示例可在[spanner-change-watcher-samples/samples . Java](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/samples/spanner-change-watcher-samples/src/main/java/com/google/cloud/spanner/watcher/sample/Samples.java)文件中找到，示例用于:

*   错误处理
*   配置轮询间隔
*   使用自定义提交时间戳存储库
*   使用[分片](https://cloud.google.com/spanner/docs/whitepapers/optimizing-schema-design#anti-pattern_timestamp_ordering)允许对提交时间戳列进行索引，以优化轮询