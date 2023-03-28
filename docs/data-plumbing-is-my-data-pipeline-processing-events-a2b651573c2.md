# 数据管道—我的数据管道在处理事件吗？

> 原文：<https://medium.com/google-cloud/data-plumbing-is-my-data-pipeline-processing-events-a2b651573c2?source=collection_archive---------0----------------------->

这个例子展示了如何在 GCP 用谷歌云完全管理的企业级 cron 作业调度器来实现一个探测，以[检查一个端到端管道是否工作](https://twitter.com/polleyg/status/1125651245035712512)。

![](img/4952c13cc7df1673ecb33275d2b6a22b.png)

想象一下，您正在构建一个令人惊叹但复杂的多步骤管道来实现您的[流分析](https://cloud.google.com/solutions/big-data/stream-analytics/)用例，涉及许多 GCP [发布订阅](https://cloud.google.com/pubsub/)主题和[数据流](https://cloud.google.com/dataflow/)流作业来处理事件，此外，管道可能会与其他服务交互以丰富事件并写入外部接收器。

您现在即将上线，并开始打开“水龙头”，让事件在管道中流动。当涉及到监控您的管道时，您可以使用强大的指标来单独监控您的 [Pubsub 主题](https://cloud.google.com/pubsub/docs/monitoring)或您的[数据流作业](https://cloud.google.com/dataflow/docs/guides/using-stackdriver-monitoring)，但是当您想要以整体的方式端到端地检查管道的状态时，会发生什么呢？

*   我可以不仅仅监视第一个主题和最后一个数据流作业，并相应地生成警报吗？可以，但我们仍需要验证缺少事件是因为数据源没有发送事件，还是因为接受新事件时出现问题，最后一个流式作业也可能没有接收到任何内容，因为卡在两个主题之间的中间作业中，同时用一些元数据丰富事件。更糟糕的是，您将无法提前识别错误，直到您收到新的事件，这些事件将引发错误，但为时已晚！
*   我可以使用某种探测/健康检查注入一些虚拟事件吗？你当然可以！您希望通过管道端到端地处理有效事件，然后可以将这些事件插入到管道末端的接收器中，以便定期清除。如果探测器未能注入事件或者死信接收器中没有接收到任何内容，则会生成警报。

![](img/d34f653164efba215afbc79281722ca4.png)

让我们使用下面的 GCP **无服务器**服务来演示如何使用云调度程序实现探测:

*   Cloud Scheduler 作为探针来生成在我们的 Pubsub 主题中注入的事件
*   GCP [Pubsub](https://cloud.google.com/pubsub/) 话题为短信服务
*   GCP [数据流](https://cloud.google.com/dataflow/)模板来处理我们的事件并将结果发送给大查询
*   [GCP BigQuery](https://cloud.google.com/bigquery/) ，谷歌快速、高度可扩展、经济高效、完全托管的云数据仓库，用于分析，作为汇

## 例子

我们将用一个简单的例子来演示这个概念，使用来自你的 GCP 项目的[云外壳](https://cloud.google.com/shell/)(也可以通过 GCP UI 控制台来完成)中的`gcloud commands`，并利用[数据流模板](https://github.com/GoogleCloudPlatform/DataflowTemplates)来简化。

我们将注入下面简单的 JSON 消息:

```
{“id”:”test”,”data”:”Testing probe”}
```

0.在开始之前，确保您拥有正确的权限，并且在项目中为所需的组件(Pubsub、BigQuery、GCS 和 Dataflow)启用了 API

1.  让我们创建我们的 Pubsub 主题:

```
gcloud pubsub topics create **[TOPIC_NAME]**
```

2.我们继续使用 BigQuery，创建输出表，实际上这可以是我们监控的主题或任何其他可用的接收器。我们使用 BigQuery 来方便地查询结果，在这种情况下，我们需要创建一个表，其模式与我们将从探针注入的消息兼容。使用 [*bq*](https://cloud.google.com/bigquery/docs/bq-command-line-tool) 命令行工具:

```
bq --location=**[LOCATION]** mk --dataset **[PROJECT_ID]**:**[DATASET]**
bq mk --table **[PROJECT_ID]**:**[DATASET]**.**[TABLE]** id:STRING,data:STRING
```

3.我们现在可以提交一个[数据流模板](https://cloud.google.com/dataflow/docs/guides/templates/overview)来从我们的主题中提取消息，并将它们插入到 BigQuery 中。对于这个例子，我们将使用一个简单的流作业[**PubSubtoBigQuery**](https://cloud.google.com/dataflow/docs/guides/templates/provided-templates#cloud-pubsub-to-bigquery)，它可以用作基线并相应地扩展。这个模板接收 JSON 消息，执行 UDF 来转换消息，并将它们插入 BigQuery 中，所有这些都没有编写任何 Java 或 Python 代码，而是使用了我们自己的参数，太棒了！

```
gcloud dataflow jobs run **[JOB_NAME]** \
    --gcs-location gs://dataflow-templates/latest/PubSub_to_BigQuery \
    --parameters \
inputTopic=projects/**[PROJECT_ID]**/topics/**[TOPIC]**,\
outputTableSpec=**[PROJECT_ID]**:**[DATASET]**.**[TABLE_NAME]**
```

4.最后，我们需要用 GCP 调度程序创建我们的探针，很简单！

```
gcloud beta scheduler jobs create pubsub **[JOB]** --schedule="* * * * *" --topic=**[****TOPIC]** --message-body='{“id”:”test”,”data”:”Testing probe”}'
```

我们使用 [cron 格式](https://en.wikipedia.org/wiki/Cron#Overview)来调度我们的探测，在我们的例子中是按分钟调度的，但是我们可以很容易地用`"0 * * * *"`把它改成类似每小时。

既然我们已经安排了探测，我们可以验证我们确实每分钟都在接收来自探测器的消息。下面的查询应该返回探测器在管道中注入的每条消息的行。

```
bq query --use_legacy_sql=false 'SELECT * FROM `**[DATASET]**.**[TABLE_NAME]**` LIMIT 1000'
```

## 结论

这个使用无服务器服务的简单示例展示了如何实现一个探针来将消息注入到我们的管道中，以验证是否完全正常工作。希望这可以让人们了解如何为他们自己的特定管道实现探测，但是请记住，像注入带有更新时间戳的消息这样的复杂用例并不被直接支持，因为云调度器目前不支持动态消息。