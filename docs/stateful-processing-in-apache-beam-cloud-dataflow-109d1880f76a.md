# Apache Beam/Cloud 数据流中的状态处理

> 原文：<https://medium.com/google-cloud/stateful-processing-in-apache-beam-cloud-dataflow-109d1880f76a?source=collection_archive---------0----------------------->

![](img/da8b7e8d5558ad4be52572eb598ef316.png)

[Apache Beam](https://beam.apache.org/get-started/beam-overview/) 是一款开源软件，用于构建大规模批处理和流数据并行处理管道，可以在各种分布式后端上运行——在本文中，我将重点介绍[谷歌云数据流](https://cloud.google.com/dataflow)。经常会出现需要有状态处理的用例，在本文中，我的目标是通过提供一个示例用例、注意事项和代码来解释如何在分布式数据处理系统中维护状态，从而利用 Apache Beam 构建 [*有状态处理。*](https://beam.apache.org/blog/stateful-processing/)

# 示例使用案例

一家出租车公司希望有一个仪表板，显示车队中所有出租车司机的状态，数据不超过 1 分钟。公司想知道有多少出租车是可用的、繁忙的、空闲的。该数据将被表示为一个条形图，该条形图根据多少辆出租车处于何种状态来改变值。

对于此示例，让我们考虑以下要求:

*   同一辆出租车可以在一天中多次改变状态，因此管道必须维护状态以跟踪每辆出租车的最新状态。
*   当出租车改变状态时，事件被触发，并以以下格式发布到 [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/docs/overview) :

*   所有出租车的 **taxi_data** 将通过云发布/订阅读取，输出将发布到发布/订阅主题。输出将是一个元组，其中每个值代表特定州的出租车数量。 *(int:忙，int:可用，int:空闲)*

# 考虑

**在编写有状态的射束管道时，考虑输入源、工作是如何分配的以及您的数据可能有多晚是很重要的。**这些因素在可用数据的类型、不同的处理优化以及输出数据的准确性方面起着至关重要的作用。

# 源输入

在这种情况下，输入源是云发布/订阅。如果没有在订阅上启用[消息排序](https://cloud.google.com/pubsub/docs/ordering)，云发布/订阅**无法**保证数据发送到管道的顺序。这本身会导致争用情况，即旧数据可能在新数据之后进入管道，这可能导致旧数据覆盖当前状态，并导致不准确的结果。

# 工作分配

第二个考虑因素是如何分配工作，这一点至关重要，因为云数据流在处理数据时只有两个保证:

1.  属于同一个键的所有工作都将分配给同一个工人
2.  每个工作包都将在一个线程上处理

当使用 Cloud Dataflow 的[流媒体引擎](https://cloud.google.com/dataflow/docs/guides/deploying-a-pipeline#streaming-engine)服务并通过 Java 的 [PubsubIO](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/io/gcp/pubsub/PubsubIO.html) 或 Pythons 的 [pubsub](https://beam.apache.org/releases/pydoc/current/apache_beam.io.gcp.pubsub.html) 模块从 Cloud Pub/Sub 读取时，融合到读取阶段的所有阶段都将根据用户工作人员的最大数量设置一组键。在这种情况下，有 1024 个随机生成的密钥。每个工人，一个[谷歌计算引擎](https://cloud.google.com/compute)实例，将被给予一个键范围，它负责处理，以确保属于一个键的所有工作都将进入同一个工人。

包是任意工作项的集合，可以包含来自多个键的数据。一个密钥永远不会出现在两个平行的包中。例如，worker 1 处理键[a，b，c，d],因此它可能收到一个包含键(a，c)的工作的包，然后它收到的下一个包可能包含(d)的工作，等等。每个包都保证在一个线程上处理。对于这个用例来说，这是一个重要的考虑因素，因为这意味着不仅数据的接收没有顺序，而且数据的处理也是不确定的。

# 数据延迟

第三个考虑因素是数据在不再有用之前可以有多晚——这取决于用例。对于当前的需求，这是一个重要的因素，因为目标是为每个 **taxi_id 保存最新的 **taxi_status** 。**

现在是时候将这些考虑转化为设计原则，以确保管道能够满足上述用例需求。

# 构建有状态管道

## 阶段 1:从发布/订阅订阅中读取并设置用户定义的密钥

Stage [1]从 Python 中的 Beam 的 [PubsubIO 连接器读入数据，然后将输出](https://beam.apache.org/releases/pydoc/current/apache_beam.io.gcp.pubsub.html) [PCollection](https://beam.apache.org/documentation/programming-guide/#pcollections) (消息)传递到 **SetTaxiKeyFn** ( [beam。DoFn](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=dofn#apache_beam.transforms.core.DoFn))[pt-1–2]，它做两件事。它首先将数据转换成一个本地字典对象，该对象允许在管道的后续阶段中更容易地解析和访问数据字段。其次，它为我们的数据定义了键。[键](https://beam.apache.org/documentation/transforms/python/elementwise/keys/)是元组形式的键值对，在这种情况下，DoFn 将 **taxi_id** 设置为键。

当从 PubsubIO 读取数据时，有 1024 个随机生成的哈希键。然而，由于用例需要跟踪与每辆出租车相关的所有数据，我们需要保证与出租车相关的所有数据都归一个工人所有。通过将每辆出租车的唯一标识符 **taxi_id** 设置为密钥，我们现在可以保证与每辆出租车相关的所有数据都将由同一个工作人员发送和处理。

## 阶段 2:有状态 DoFn 和窗口

现在，既然我们已经定义了数据将如何分布和处理的逻辑分组，下一步，stage[3]**statefullpardofn**(beam。DoFn)是处理存储状态和更新状态逻辑。由于该处理是针对每个键执行的，因此使用 [BagStateSpec](https://github.com/apache/beam/blob/92386d781b5d502c4ea47a0894b72ca57854553d/sdks/python/apache_beam/transforms/userstate.py#L82) 提供了一种维护每个键的状态的方法。默认情况下，有状态 DoFn 在键和窗口的基础上工作，但是因为我们使用了 [GlobalWindows](https://beam.apache.org/documentation/programming-guide/#single-global-window) ，它在每个键上工作，并且存在于 DoFn 实例中，每个 worker 只创建一次。下一部分是现在定义一个窗口阶段[4] GlobalWindow，以允许我们在后续阶段中维护一个有状态累积。

[BagStateSpec](https://beam.apache.org/releases/pydoc/2.6.0/apache_beam.transforms.userstate.html#apache_beam.transforms.userstate.BagStateSpec) 是一个有状态的单元，它为每个键创建一次，并为我们提供存储每个键的有状态信息的能力。通过创建一个 **TAXI_STATE** BagStateSpec，我们现在可以存储与每个 **taxi_id** 相关的某个州。对于当前用例，我们的 **TAXI_STATE** 包将最多只为每辆出租车存储一个元素，其中该元素是 **taxi_id** 的最新状态 **taxi_status** 。由于 BagStateSpec 每个键创建一次，因此我们的管道中创建的包的数量=唯一的 **taxi_id** 的数量。包的生命周期取决于它所在窗口的持续时间。

现在我们有了一种方法来存储一个 **taxi_id** 的最近状态，下一部分是实现根据用例更新它的逻辑。在这种情况下，如果数据较新且状态不同，我们只想更新 **taxi_id** 的状态。由于 **taxi_data** 有一个时间戳，我们可以将它和出租车的状态存储在包中，并使用它来检查传入数据的时间戳和状态。如果数据较新并显示状态改变，我们可以更新箱包。

由于与我们的出租车相关的所有数据都分布在所有工人身上，因此需要一种方法来将元素组合在一起— [CombineGlobally()](https://beam.apache.org/documentation/transforms/python/aggregation/combineglobally/) 。该函数组合、聚合和累加所有值，忽略分布在所有机器上的前一级 PCollections 中的 key 元素。考虑这一点很重要，因为支配规则是一个键的所有数据都将进入一台机器，这意味着现在我们的累积步骤 CombineGlobally()将全部发生在一个 worker 中。为了澄清，所有的元素将由同一个工人处理，但是所有的累加器将被发送给一个工人。很多工人会执行*。create_accumulator()* 和*。add_input()* ，并行发生。在 CombineGlobally 中，所有的累加器都被发送给一个工人来运行*。merge_accumulators()* 并提取单个输出。另一种帮助减轻所有工作到一个工人机器的方法是使用*。withHotKeyFanout()* ，只能与 *CombinePerKey()* 函数一起使用。

为了使这一步尽可能快，将每个状态的值相加会更容易，因为累加阶段不关心 **taxi_id** 。我们可以将输出 PCollection 转换为相同的格式， *(int:busy，int:available，int:idle)* ，然后将相同位置的所有值相加，得到每个州有多少辆出租车的总运行计数。

例如，如果包当前包含 **taxi_id=5** 的“忙碌”状态，并且 DoFn 接收到显示 **taxi_id=5** 现在“空闲”的较新的数据，我们应该将包和 now -1 从处于“忙碌”的出租车和+1 更新为处于“空闲”的出租车。我们的输出元组现在应该看起来像 *(-1，0，1)* 。这处理了从先前的状态计数中删除出租车的逻辑。

如果对于 **taxi_id=5** 没有任何先前记录的状态，那么我们的输出元组将类似于 *(0，0，1)* 。如果没有任何状态变化，我们可以简单地输出 *(0，0，0)* ，因为没有任何状态变化。

需求的下一部分提到，出租车公司想要每 N 秒触发一次窗口的<= 1 minute of lateness. By explicitly defining a GlobalWindow in stage [4], we can use an [AfterProcessingTime()](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.trigger.html?highlight=afterprocessingtime#apache_beam.transforms.trigger.AfterProcessingTime) 数据，在这个用例中，这是 60 秒。

既然我们设置了触发器，我们还必须定义一个累积模式，因为触发器可以触发多次。[累积模式](https://beam.apache.org/documentation/programming-guide/#window-accumulation-modes)定义窗口*如何累积*或*在触发时如何丢弃*来自不同窗格的数据。通过将其设置为 ACCUMULATING，我们将从所有窗格中累积数据，而不管它被触发了多少次。我们需要使用累加，因为 CombineGlobally()步骤中状态的生命周期依赖于所有先前进入管道的数据。

到目前为止，我们的管道现在从 Cloud Pub/Sub 读取 **taxi_data** ，使用用户定义的键来组织数据，存储每辆出租车的最新 **taxi_status** ，并设置一个窗口策略来将该数据发送到管道的下一阶段。

> 注意:窗口将被忽略，直到一个洗牌步骤或融合阶段之间的转换(即 GroupBy*、Combine*)，这将涉及与流后端的检查点。

## 阶段 3:有状态累积和发布输出到云发布/订阅

在完成了所有前面的阶段之后，此时的 PCollection 是具有以下格式的元组列表: *(int:busy，int:available，int:idle)* 。我们现在可以使用 *CombineGlobally()* 函数来聚合来自所有工人的所有键，因为输出应该是每个 **taxi_status** 中出租车总数的累积值。

在实现 [CombineFn](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=get_accumulator#apache_beam.transforms.core.CombineFn) 时，需要实现四个主要方法。

[create _ accumulator()](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=get_accumulator#apache_beam.transforms.core.CombineFn.create_accumulator)=>初始累加器状态

*   这将保存每个 **taxi_status** 中出租车总数的累计值

[add_input()](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=get_accumulator#apache_beam.transforms.core.CombineFn.add_input) = >定义更新累加器的逻辑

*   在这种情况下，此状态下的逻辑只是将输入添加到当前累加器状态。

[merge _ accumulators()](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=get_accumulator#apache_beam.transforms.core.CombineFn.merge_accumulators)=>合并跨工作线程的累加器

*   组合使用分而治之的技术。每个工作者将有一个累加器，该状态将被传递给被分配了该键的一个工作者。这将导致所有值的最终累加。

[extract_output()](https://beam.apache.org/releases/pydoc/current/apache_beam.transforms.core.html?highlight=get_accumulator#apache_beam.transforms.core.CombineFn.extract_output) = >调用来检索合并的最终结果。

*   调用此函数以返回最终值作为其输出 PCollection。

在我们积累了多少辆出租车处于何种 **taxi_status** 状态的答案之后，下一步就是对数据进行格式化和编码，以便将其发布到 PubsubIO。

阶段[7]是每隔一分钟将数据发布到 Pub/Sub 主题以供出租车公司下游使用的最后阶段。

请随时留下评论并检查整个代码！