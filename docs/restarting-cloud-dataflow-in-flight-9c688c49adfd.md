# 正在重新启动/更新云数据流

> 原文：<https://medium.com/google-cloud/restarting-cloud-dataflow-in-flight-9c688c49adfd?source=collection_archive---------0----------------------->

我不容易被魔术打动…当你每天用谷歌云平台工作时，你会习惯魔术。但今天，我对云数据流服务印象深刻，它能够在不停止工作的情况下更新您的工作。

![](img/e293d04fa56373a0fd5c04f47c65b291.png)

[https://unsplash.com/search/stream?photo=8X1-pcDF8l0](https://unsplash.com/search/stream?photo=8X1-pcDF8l0)

Cloud Dataflow 是 Google 的托管执行引擎，用于运行用 [Apache Beam](https://beam.apache.org/) 编写的数据处理工作流。我们使用 Beam 模型的主要原因是能够编写一次管道，然后运行**流**或**批处理**。但是当你深入研究这个模型时，真正的力量在于它处理事件时间和不同类型窗口的能力。

“会话窗口”就是一个例子:通过管道中的一行代码，您可以从您的数据创建会话。上面的代码以 20 分钟的间隔将您的数据分组。

![](img/26be8b95fe641805e25fba911f303cac.png)

[https://unsplash.com/search/dam?photo=DMfSpSrhF-k](https://unsplash.com/search/dam?photo=DMfSpSrhF-k)

有了这样一个定义窗口的强大模型，你可以想象更新一个正在运行的" **streaming** "数据流，甚至恢复故障都不是一件小事。你就是不能拔掉插头重新启动…你如何处理所有已经确认但尚未写入的运行中的数据？！事实证明，云数据流支持更新。

但是今天，我们使用这种机制给了一个失败的数据流一脚，这样它就可以从由于 BigQuery 表没有及时创建而导致的故障中恢复过来。数据流的缓冲区中有大约 8 小时的数据，我们不想丢失。事件得到了确认，所以它们来自 PubSub，但也没有写入 BigQuery，它们在数据流管道中。

```
java -jar pipeline/build/libs/pipeline-ingress-1.0.jar \
        --project=my-project \
        --zone=us-central1-f \
        --streaming=true \
        --stagingLocation=gs://my-bucket/tmp/dataflow/staging/ \
        --runner=DataflowPipelineRunner \
        --numWorkers=1 \
        --workerMachineType=n1-standard-2 \
        --jobName=**ingresspipeline-1216151020** \
        **--update**
```

在数据流上使用 update 标志可以将配置切换到特殊模式，确保没有数据丢失。它停止正在运行的数据流，确保已经被确认的和在管道中的数据被持久化到磁盘(例如未完成的会话)并将磁盘重新连接到新数据流并继续运行。

没有什么比一次真正的紧急事件带来的好结果更能提高人们对产品的信心了。唯一的缺点是:正常运行时间计数器从 200 多天重置为 0。但这对于不丢失任何数据来说是一个很小的代价。

[](https://cloud.google.com/dataflow/pipelines/updating-a-pipeline) [## 更新现有管道|云数据流文档|谷歌云平台

### “运行中”的数据仍将由新管道中的转换进行处理；但是，其他转换…

cloud.google.com](https://cloud.google.com/dataflow/pipelines/updating-a-pipeline)