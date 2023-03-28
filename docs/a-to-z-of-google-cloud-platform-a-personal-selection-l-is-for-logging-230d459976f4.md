# Google 云平台的 a 到 Z 个人选择— L 代表登录

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-l-is-for-logging-230d459976f4?source=collection_archive---------7----------------------->

收集和分析日志有很多原因，通常是因为

*   寻找应用程序异常
*   审核日志
*   趋势分析
*   还有很多其他原因(现在我不会花太多时间在这上面)

尽管监视与日志记录密切相关，并且经常携手并进，但我将在下一篇文章中讨论监视，因此您可以将此视为两部分中的第一部分。

希望你收集日志是因为你要用它们做一些事情，如果是这样，你会很高兴知道 GCP 有办法帮助你收集日志并分析它们，不管你出于什么原因收集它们。如果你不确定为什么要收集它们，那么就使用 [nearline](https://cloud.google.com/storage/docs/nearline) 来存储它们，直到你确定。

**收集&存储应用日志**

日志收集需要一种从生成日志的地方获取日志并将它们移动到其他地方的方法

如果你没有在 GCP 上运行你的应用程序，那么你可以很容易地将你的日志传输到 GCS 或者 Big Query，或者 pub/sub，所以这篇文章的大部分内容仍然适用，特别是关于分析和处理日志的最后一点。我将主要关注假设您的应用程序运行在 GCP 上，所以做一些您需要的心理调整。这是一个关于 GCP 的系列，所以我不想偏离太远，因为这里的一些帖子已经很长了。)

GCP 有一个[云日志服务](https://cloud.google.com/logging/docs/)，它收集并存储来自应用程序和服务的日志。

当在 GCE 上运行应用程序时，它支持开箱即用的一组[通用应用程序](https://cloud.google.com/logging/docs/view/logs_index#custom_types)。如果您使用计算引擎，则安装[日志代理](https://cloud.google.com/logging/docs/agent/)。该代理负责将日志从常见的第三方应用程序传输到云日志。

为了将日志条目从您自己的应用程序写入云日志服务，GCP 提供了一个[日志 API](https://cloud.google.com/logging/docs/api/introduction_v2) 。您还可以使用 API 将日志导出到其他服务。

除了从应用程序收集日志，GCP 还收集[活动日志](https://cloud.google.com/compute/docs/activity-logs)。这些跟踪影响项目的某些事件，例如 API 调用和系统事件。

云日志服务存储数据 [30 天](https://cloud.google.com/logging/quota-policy)，因此对于更长期的存储，您可以使用[日志导出](https://cloud.google.com/logging/docs/export/)服务导出到 GCS、Big Query，并通过 pubsub 传输到其他服务，如[数据流](https://cloud.google.com/dataflow/what-is-google-cloud-dataflow)或 [Dataproc](https://cloud.google.com/dataproc/overview) 。

**日志&审计**

收集日志文件的一个重要原因是提供审计跟踪。GCP 提供[审计日志](https://cloud.google.com/logging/docs/audit/)，这实际上是捕获管理活动和数据访问的两个日志流，由 GCP 服务生成。与其他日志类型一样，这些日志可以导出，以便长期存储。

您可能希望限制谁有权访问这些日志。这可以通过使用 IAM 角色将[访问控制应用到日志](https://cloud.google.com/logging/docs/access-control)来实现

**分析&处理日志**

这样你就有了已经导出的日志。如果您已经[直接导出到 BigQuery](https://cloud.google.com/logging/docs/export/using_exported_logs#log_entries_in_google_bigquery) 中，那么您可以通过几个查询开始询问日志问题。这是如此简单却又如此强大！

您可以使用日志导出服务[导出到云存储](https://cloud.google.com/logging/docs/export/using_exported_logs#log_entries_in_google_cloud_storage)，但实际上您是在收集日志来对它们做一些事情，所以除非您需要对它们做一些 ETL 类型的处理，否则将日志保存在云存储上可能是有意义的，直接导出到 BigQuery 可能是有意义的。

但是对于一个更复杂的过程，使用[数据流](https://cloud.google.com/dataflow/what-is-google-cloud-dataflow)或[数据块](https://cloud.google.com/dataproc/overview)可能是需要的。如果你的应用程序运行在 GCP 上，那么你可以将日志导出到 GCS，并从那里使用 Dataflow 或 Dataproc 进行处理。GCP 有一个在这个场景中使用数据流的很好的[演练，我没有什么有价值的东西要补充，因为这是一个非常全面的演练。](https://cloud.google.com/solutions/processing-logs-at-scale-using-dataflow)

也许你想将日志直接流式传输到数据流，在这种情况下，你可以使用导出服务将日志[导出到 pub/sub](https://cloud.google.com/logging/docs/export/using_exported_logs#log_entries_in_google_pubsub_topics),或者如果在本地或另一个云上运行，只将日志从源应用程序流式传输到 pub/sub。然后，您可以将数据从发布/订阅传输到您的目标，如数据流。

**日志和监控**

正如我前面提到的，应用程序通常都有相关的日志文件。这些日志将记录各种应用程序事件。您可以通过对相同类型/错误的消息进行计数来跟踪它们，然后通过使用这些基于[日志的指标](https://cloud.google.com/logging/docs/view/logs_based_metrics)您可以利用 GCP 的完全托管监控解决方案云监控(我将在本系列的下一篇文章中讨论)来包括从您的应用发出的各种日志中整理的基于日志的指标，以创建图表和[警报策略](https://www.youtube.com/watch?v=KGlP6m0JbYw&feature=youtu.be)

**可视化日志数据**

你可能也想想象一下这些，猜猜是什么让 GCP 变得如此简单。您可以使用 [Datalab](https://cloud.google.com/datalab/getting-started) 来分析存储在 BigQuery 和云存储中的数据。如果您已经将日志导出到其中任何一个，那么您可以使用 Datalab 通过一些可视化来增强您的查询。数据实验室建立在 Jupyter 笔记本电脑上，有一个庞大的生态系统来帮助您起步

在本系列下周的文章中，我将在此基础上讨论监控。