# 在 Google Cloud 数据流和 Apache Beam 上构建实时量化交易引擎

> 原文：<https://medium.com/google-cloud/building-a-real-time-quant-trading-engine-on-google-cloud-dataflow-and-apache-beam-841a909d2c12?source=collection_archive---------0----------------------->

## GCP 数据流的一个实际用例

![](img/67afd429ee171d36edb46da9765d723c.png)

谷歌云拥有完全托管的服务，允许最终用户为其分析需求构建大数据管道。其中一个叫做数据流。它允许开发人员基于 Apache Beam SDK 构建数据管道。它是完全托管的，因为 gCloud 负责自动扩展分布式工作资源，以及在可用节点之间重新分配工作负载。与 Apache Beam 一起，它为批处理和流数据提供了统一的编程模型。它还可以方便地与其他 gCloud 产品集成，如 Pub/Sub 和 BigQuery。

在这篇文章中，我们将构建一个数据管道，分析从 gCloud Pub/Sub 流出的实时股票报价点数据，通过配对相关交易算法运行它们，并将交易信号输出到 Pub/Sub 上以供执行。

免责声明:这篇文章旨在实验在数据流上建立实时数据分析，而不是教如何编码量化交易策略。如果没有适当的风险管理，不要在真实交易中使用这种交易策略。

在详细介绍管道之前，先介绍一下交易策略的背景。我们的交易算法是基于这样的假设，即相似的股票长期来看是一致的。例如，像谷歌、苹果、facebook 这样的大型科技股价格以一种相关的方式移动。如果出于某种原因，它们不再相关，这可能是由不寻常的市场事件引起的。我们有机会打开一对股票的多头/空头头寸，并在相关性回归均值时平仓。这是一个简单的交易股票对的均值回归策略。

例如，使用滑动窗口，我们可以每隔 Y 分钟计算股票 A 和 B 的最近 X 点价格的相关性。在正常的市场条件下，A 和 B 应该是正相关的，这意味着它们的相关性应该接近 1。如果由于任何原因，它们的相关性变为负值，并低于某个阈值，这就是我们的交易信号，即开仓做多股票 A 和做空股票 B。我们将继续监控 A 和 B，直到它们再次变为正相关，然后我们可以平仓。正如任何多空交易策略一样，这对组合也可能以相反的方式运行。严格的风险管理触发机制已经到位，以防止不受约束的损失，但这与本文的重点无关。

我们基于数据流的交易管道看起来像这样

![](img/dca9891088015a52648ac58ab31e23dc.png)

Apache Beam SDK 允许我们使用统一的编程模型在 gCloud 数据流上构建数据管道。它的核心是[p 集合](https://beam.apache.org/documentation/sdks/javadoc/2.1.0/org/apache/beam/sdk/values/PCollection.html)的概念。PCollection 是数据集的不可变容器。它是管道中每个变压器的输入和输出。它可以是有界的，这意味着数据集的大小是已知的，例如来自文本文件的文件 IO，来自数据库查询的 jdbc IO。它也可以是无限的，这是流数据的情况，例如来自队列外的消息的 Kafka IO。

Beam SDK 还提供了几个通用的[转换](https://beam.apache.org/documentation/programming-guide/#applying-transforms)，可以与[集合](https://beam.apache.org/documentation/sdks/javadoc/2.1.0/org/apache/beam/sdk/values/PCollection.html)一起工作。它基于 Map/Reduce 编程模型。在这篇文章中，我们将使用

1.  [ParDo](https://beam.apache.org/documentation/sdks/javadoc/2.0.0/org/apache/beam/sdk/transforms/ParDo.html) —用于并行处理的通用映射变换，可产生 0、1 或 N 个输出。在本练习中，我们使用 ParDo 将传入的单个分笔成交点数据流分成成对的数据流，以便可以动态计算它们的相关性。如果有 10 个滴答流，该步骤将它们配对并输出 90 个数据流。我们还使用它向每个分笔成交点数据添加时间戳，以便它们可以在同一个窗口中分组。
2.  [CoGroupByKey](https://cloud.google.com/dataflow/java-sdk/JavaDoc/com/google/cloud/dataflow/sdk/transforms/join/CoGroupByKey) —通过同一个键连接两个或多个键/值 p 集合。在我们的练习中，我们使用 CoGroupByKey 按股票代码将分笔成交点数据按指定对分组。
3.  [过滤器](https://beam.apache.org/documentation/sdks/javadoc/2.2.0/org/apache/beam/sdk/transforms/Filter.html) —使用 lambda or 函数过滤输入 p 集合
4.  [映射](https://beam.apache.org/documentation/sdks/javadoc/2.0.0/org/apache/beam/sdk/transforms/MapElements.html) — 1 对 1 将输入 p 集合转换为每个数据点的输出
5.  [WindowInto](https://cloud.google.com/dataflow/model/windowing) —对于无限制的分笔成交点流数据，我们使用滑动窗口将每 1 分钟 10 分钟的分笔成交点数据组合在一起。计算每个窗口数据的相关性

现在让我们来看看一些代码。

首先，我们定义了股票符号和成对组合的范围。让我们初始化我们的射束管道实例

然后，对于每只股票，我们都有以“input_ <symbol>”形式通过 gCloud Pub/Sub 流入的主题分笔成交点市场数据。每个分笔成交点数据只是一个时间戳和价格的元组。然后，我们通过一个接一个地应用转换来构建管道。</symbol>

这里有很多东西需要打开。让我们一次穿过一根管子。

1.  从 gCloud Pub/Sub 中读取宇宙中每只股票的分笔成交点数据流。我们假设 tick 数据以“input_ <symbol>的形式发布到主题上，例如 input_goog 有可供 Google 使用的 tick 数据。分笔成交点数据是时间戳和价格的元组。由于 Python 的动态类型，Beam 的 Python SDK 使用类型提示来确保运行时类型安全。在这种情况下，我们告诉运行时从发布/订阅中读取的数据是二进制类型的</symbol>

2.然后，我们应用映射转换和过滤器将字节解码成字符串，并过滤掉无效数据(如果有的话)

3.然后，我们应用 ParDo 变换为每个数据点添加时间戳。时间戳与分笔成交点数据的时间戳相同。这是使滑动窗口在无边界的 p 集合中工作所必需的。AddTimestampDoFn 扩展射束。DoFn 和覆盖过程功能。

4.然后，我们对数据流应用滑动窗口。对于每个符号的 10 分钟的分笔成交点数据，每分钟计算一次滑动窗口。该窗口影响流水线下游的任何关键操作的分组

5.现在到了利息部分。我们应用另一个 ParDo 将单个分笔成交点数据流配对，并输出多个配对数据流。我们的 ParDo 函数与步骤 3 中的 add timestamp ParDo 具有相同的模板。注意在输出流中，键不再是单个股票符号，而是一对符号

6.既然我们已经对输入流进行了配对，我们可以通过配对键对值进行分组，这样每个符号的所有市场数据都在值列表中。在这种情况下，我们的键是(' goog '，' aapl '，' fb ')或(' goog '，' fb ')中的一个。这些值是该对的分笔成交点数据的迭代器。

7.现在我们准备在滑动窗口中计算成对相关。我们从分笔成交点数据中构建熊猫序列，并计算每一对的相关性。输出是密钥对和相关系数元组。

8.管道中的最后一步是为负相关性低于阈值的对添加过滤器，并将该对作为交易信号输出到 gCloud Pub/Sub。同样，我们使用类型提示来确保 Python 使用二进制类型将数据发布到发布/订阅上。

现在，如果我们想在谷歌云数据流中运行我们令人敬畏的 quant 交易管道，我们可以按照这个[快速启动](https://cloud.google.com/dataflow/docs/quickstarts/quickstart-python)来设置我们的 GCP 项目，认证和存储。然后我们需要做的就是使用这个命令在数据流上运行管道

```
python -m correlated_trading.trading_pipe \
  --project $PROJECT \
  --runner DataflowRunner \
  --staging_location $BUCKET/staging \
  --temp_location $BUCKET/temp \--input_mode stream \--input_topic tick_data_input \
  --output_topic trading_signal
```

总之，Google Cloud Dataflow 和 Apache Beam 一起为构建数据管道和转换提供了一个富有表现力的 API。Apache Beam 编程模型在习惯转换和窗口语义方面有一些学习曲线。它迫使用户考虑数据管道的每个阶段和输入/输出数据类型，在我们看来这是一件好事。谷歌云在并行处理、分配工作单元和自动扩展方面管理着许多神奇的东西。Dataflow 与 BigQuery、Pub/Sub 和机器学习等其他 GCP 产品无缝集成，这使得生态系统对于大数据分析来说非常强大。

一如既往，你可以在 [Cloudbox Labs github](https://github.com/cloudboxlabs/blog-code/tree/master/correlation_trading) 上找到这篇文章中讨论的完整代码。