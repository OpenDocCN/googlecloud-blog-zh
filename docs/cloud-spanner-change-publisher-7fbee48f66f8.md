# 云扳手更改发布器

> 原文：<https://medium.com/google-cloud/cloud-spanner-change-publisher-7fbee48f66f8?source=collection_archive---------2----------------------->

[Google Cloud Spanner](https://cloud.google.com/spanner) 是一个全面管理、可扩展的关系数据库服务，用于区域和全球应用数据。它是第一个可扩展的、企业级的、全球分布的、高度一致的数据库服务，专为云构建，将关系数据库结构的优势与非关系水平扩展相结合。

![](img/16400067b46aef8a444b688bcf1f44a4.png)

谷歌云扳手

多个应用程序和服务可以同时与同一个 Cloud Spanner 数据库交互，常见的模式是一个服务基于某些事件触发其他服务，比如在数据库中插入或更新记录。本文描述了开源的 [spanner-change-publisher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-publisher) 服务，它可以作为一个独立的应用程序运行，将云 spanner 数据库中的数据更改发布到 Pubsub。

开源项目还包含 [spanner-change-watcher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-watcher) 库，该库可以集成到现有的 Java 应用程序中，以将数据更改事件直接交付给该应用程序，而不是将这些事件发布到 Pubsub。关于这项服务的介绍可以在[这里](/@knutolavloite/cloud-spanner-change-watcher-b77ca036459c)找到。

# 先决条件和限制

*   Spanner Change Publisher 使用[提交时间戳](https://cloud.google.com/spanner/docs/commit-timestamp)来确定行何时被更新。使用这个库只能监视包含具有上次更新提交时间戳的列的表。这个专栏的名字可以自由选择。
*   该库只能检测插入和更新，因为它依赖行的提交时间戳值来检测更改。要检测删除，您应该通过在行上设置 deleted 标志来实现逻辑删除，而不是实际删除它。如果在带有一个或多个标记为 ON DELETE CASCADE 的子表的父表上执行删除，那么子表也需要相应地更新，以便由观察器选取。单独的后台任务可以使用该标志来检测稍后应该删除哪些行。
*   该库通过轮询被监视的表来监视数据库的变化。轮询间隔默认为一秒钟，可以进行配置。如果在小于轮询间隔的时间范围内对同一行有多次更改，库可能只报告最后一次更改。

# 扳手更换发布者

Spanner Change Publisher 是一种服务，它监控云 Spanner 数据库中的一个、一些或所有表的数据变化，并将检测到的数据变化发布到 Pubsub。它既可以作为现有应用程序的一部分执行，也可以作为独立的应用程序执行。Spanner Change Publisher 内置了对以 Avro 和 JSON 格式发布更改数据的支持，数据转换是可配置的，并允许您提供自己的转换器来转换您可能想要使用的任何格式。

扳手变化发布器依靠扳手变化监视器来检测数据库中的变化。建议在使用扳手更换发布器之前，对扳手更换观察器有一个基本的了解。扳手更换观察器的介绍可以在[这里](/@knutolavloite/cloud-spanner-change-watcher-b77ca036459c)找到。

对一个表的更改保证按提交时间戳的顺序发布，但 Pubsub 不保证消息按发布的顺序传递给订阅者。关于 Pubsub 消息订购的更多信息，请参见 https://cloud.google.com/pubsub/docs/ordering。

# 示例应用程序

项目[包含一个小的示例应用程序](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/samples/spanner-change-publisher-samples/src/main/java/com/google/cloud/spanner/publisher/sample/SimpleChangePublisherSample.java)，它将:

1.  创建一个包含两个表的数据库(或者使用现有的数据库，如果它已经存在的话)。
2.  创建一个扳手更改发布器，它将监视数据库中所有表的更改，并将这些更改发布到一个 Pubsub 主题。
3.  创建一个从 Pubsub 接收更改并将这些更改写入控制台的 Pubsub 订阅服务器。
4.  对云扳手数据库写一些修改。订户将更改写入控制台。

## 数据模型

该应用程序使用以下数据模型。如果您从一个空数据库开始，它会自动为您创建。

## 创建扳手变更发布者

如果您的应用程序中包含 [spanner-change-publisher](https://github.com/cloudspannerecosystem/spanner-change-watcher/tree/master/google-cloud-spanner-change-publisher) 作为依赖项，您可以从自己的应用程序中配置、创建和启动变更发布程序。示例应用程序创建了一个具有以下属性的发布者:

*   观察 Cloud Spanner 数据库中所有表(带有提交时间戳列)的变化。
*   以 JSON 格式发布更改后的数据。
*   发布对单个发布订阅主题的所有更改。每个 Pubsub 消息的属性还将包含发生更改的表的名称。Spanner Change Publisher 还支持发布对每个表的不同主题的更改。关于如何做到这一点的更多信息，请参见项目中的附加示例和方法`Builder.setTopicNameFormat(String)`的文档。

为整个数据库创建扳手变化发布器

一旦发布程序运行，对给定数据库中任何表的所有更改都将发布到 Pubsub，并将传递给发布更改的主题的订阅者。

## 从云扳手订阅更改

来自 Cloud Spanner 的更改以选定的格式发布到选定的发布订阅主题。对这些更改的订阅是使用普通的 Pubsub 订阅服务器完成的。消息数据包含实际的云扳手数据，而消息属性包含更改的元数据，例如数据库名称、表名称和提交时间戳。

下面的代码片段显示了如何订阅 JSON 格式的更改并将这些更改写入标准控制台。

订阅扳手变更发布程序的示例

## 运行示例应用程序

要执行示例应用程序，请按照下列步骤操作:

1.  [从 GitHub](https://github.com/cloudspannerecosystem/spanner-change-watcher) 克隆项目。
2.  导航到`spanner-change-watcher/samples/spanner-change-publisher-samples`文件夹。
3.  通过执行`mvn clean package`创建一个. jar。
4.  执行以下命令，其中`<instance-id>,` `<database-id>, <topic-id>`和`<subscription-id>`必须替换为实际值。该实例必须已经存在，如果数据库尚不存在，将自动创建该数据库。如果 Pubsub 主题和 Pubsub 订阅不存在，也将自动创建它们。请注意，正在使用的帐户必须拥有创建主题和订阅的权限，这样才能自动创建主题和订阅。如果服务帐户没有这些权限，您也可以手动创建主题和订阅。

```
java -cp target/spanner-change-publisher-samples.jar \
com.google.cloud.spanner.publisher.sample.SimpleChangePublisherSample \
<instance-id> <database-id> <topic-id> <subscription-id>
```

示例应用程序将启动 Spanner Change 发布者和 Pubsub 订阅者，然后将一些数据写入 Cloud Spanner。数据将被写入控制台日志。

# 扳手变化发布者独立应用程序

扳手变化发布器可以作为独立的应用程序来配置和执行，而不需要集成到现有的 Java 应用程序中。要监视的数据库、应该发布更改的主题以及其他配置选项都可以在属性文件中设置。

完整的配置示例文件可以在 GitHub 上找到。下面列出了最重要的配置参数。

```
# ----- SPANNER SETTINGS ----- #
scep.spanner.project=my-spanner-project
scep.spanner.instance=my-instance
scep.spanner.database=my-database#Credentials to use for Spanner if these differ from the default
scep.spanner.credentials=/path/to/spanner-credentials.json# Turn watching all tables in the database for changes on/off.
scep.spanner.allTables=false# A list of tables that should be excluded from monitoring.
# This may only be used in combination with allTables=true.
scep.spanner.excludedTables=TABLE1,TABLE2,TABLE3# A list of tables that should be included in the monitoring.
# This may only be used in combination with allTables=false.
scep.spanner.includedTables=TABLE1,TABLE2,TABLE3# The poll interval of the Spanner watcher.
scep.spanner.pollInterval=PT0.5S# ----- PUBSUB SETTINGS ----- #
scep.pubsub.project=my-pubsub-project# Credentials to use for Pubsub if these differ from the default.
scep.pubsub.credentials=/path/to/pubsub-credentials.json# Converter to use to convert a Spanner row to a Pubsub message.
scep.pubsub.converterFactory=com.google.cloud.spanner.publisher.SpannerToJsonFactory# Topic(s) where to publish the changes.
scep.pubsub.topicNameFormat=spanner-update-%database%-%table%
```

要使用的配置文件在启动时作为系统参数给出。还可以通过将附加配置值指定为系统参数来覆盖或设置它们。

要运行 Spanner Change Publisher 独立应用程序，请遵循以下步骤:

1.  [从 GitHub](https://github.com/cloudspannerecosystem/spanner-change-watcher) 克隆项目。
2.  导航到`spanner-change-watcher/google-cloud-spanner-change-publisher`文件夹。
3.  通过执行`mvn clean package`创建一个. jar。
4.  执行`java -Dscep.properties=/path/to/your-scep.properties -jar target/spanner-publisher.jar`，其中`path/to/your-scep.properties`是您的配置文件。

## 发布订阅主题名称格式

Pubsub 主题名称格式配置值用于确定将特定表中的更改发布到何处。配置值允许使用多个通配符将不同表中的更改发布到不同的主题。您还可以指定不带任何通配符的固定格式。这将导致所有更改都发布到同一个主题。订阅者仍然可以看到更改来自哪个表，因为完整的表名包含在 Pubsub 消息的属性中。

Pubsub 主题名称格式配置中支持的通配符有:

*   %project%:云扳手数据库的项目 id
*   %instance%:云扳手数据库的实例 id
*   %database%:云扳手数据库的数据库 id
*   %table%:云扳手数据库的表名

例如，下面的配置将发布对数据库`my-database`中的表`MyTable`的更改，以发布到名为`projects/my-pubsub-project/topics/spanner-update-my-database-MyTable`的主题。

```
scep.pubsub.project=my-pubsub-project
scep.pubsub.topicNameFormat=spanner-update-%database%-%table%
```