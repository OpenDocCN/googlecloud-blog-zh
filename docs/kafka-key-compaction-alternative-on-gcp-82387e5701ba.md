# GCP 的卡夫卡按键压缩替代方案

> 原文：<https://medium.com/google-cloud/kafka-key-compaction-alternative-on-gcp-82387e5701ba?source=collection_archive---------0----------------------->

马丁·福勒在 15 年前描述了[“事件采购”](https://martinfowler.com/eaaDev/EventSourcing.html)模式。其思想是，当系统状态的所有更改都被记录在日志中，并且任何需要状态的特定具体化的服务都可以从日志中即时计算出来时，软件系统可以更好地伸缩。例如，如果您的内容管理系统(CMS)中有一系列文档更新，您可以使用这些更新的历史记录而不是文档的最新状态来初始化内容的搜索索引。这将索引系统从 CMS 使用的生产数据库中分离出来，并降低数据库的负载。如果搜索服务设计者认为有用的话，这也可以让你搜索过去的版本。

这个想法找到了与 Apache Kafka 的完美匹配，它提供了水平可伸缩的日志存储。然而，即使是水平扩展，重新处理系统的整个历史通常也是不切实际的。这导致了 Kafka 设计中的一个关键问题:它支持压缩日志，丢弃给定键的旧条目。在本文中，我们考虑另一种实现事件源的实用方法:Firestore 监听器 API[T3。为了深入了解细节，有必要更精确地回到事件源的用例，作为状态复制问题。](https://cloud.google.com/firestore/docs/query-data/listen)

![](img/f7161ec1cefe54637f3744f7081537ae.png)

**状态复制用例**

考虑事件源的一种方式是异步状态复制。在这种模型中，如同在事件源中一样，一组分布式客户端通过跟踪任何更新来构建最终一致的中央真实源的本地副本。他们可能依赖于完整的日志，但这是实现细节。这种模式的目的是确保在内存中访问最新的真实来源副本，同时有效地通知任何更改。这一点至关重要的一个应用示例是将配置分发到分布式服务的多个实例。

**卡夫卡按键压实**

状态复制模式可以通过利用 Apache Kafka 作为有序日志来实现，保留存储的每个键的值的历史。Apache Kafka 是一个开放源码的分布式存储系统，它结合了消息中间件的实时性和数据库的几个典型特性。许多数据传输用例可以使用 Pub/Sub 或 Pub/Sub Lite 在低阻抗的 GCP 上实现，但类似数据库的用例通常可以使用其他完全托管的 GCP 服务更好地解决。

当新的客户端需要初始化它们的状态副本时，它们将从日志的开始处开始读取，将曾经发生的每个更改应用到它的本地状态，并在它们到达时继续监听未来的更新。键压缩通过定期从 backlog 中清除旧的、冗余的事件，使这种访问模式更加有效，这样新的客户机就不需要处理它们了。但是，所有新连接的客户端仍然需要读取 backlog 中的每个事件，以确定任何给定键的值。

[键压缩](https://kafka.apache.org/documentation/#compaction)在有很多消息时减少了这种访问模式的开销。当启用密钥压缩时，具有相同密钥(用户提供的字符串)的旧消息将在具有相同密钥的新消息发布后被删除。这允许客户端读取全部积压的消息来建立每个现有键的状态，然后在发布新消息时看到它们来更新它们的内部状态。

**Firestore 听众**

[Cloud Firestore](https://cloud.google.com/firestore) 是一个可扩展的、完全托管的 NoSQL 数据库。虽然它与 Kafka 有很大不同，但它完全公开了 Kafka key-compacted topic 和 [Firestore listeners](https://cloud.google.com/firestore/docs/query-data/listen) 所提供的尾随语义。虽然 Firestore 主要用于移动应用程序，但 Firestore 监听器通过比 Kafka 更高级别的 API 解决了服务之间数据复制的异步复制问题。

Firestore 监听器使客户能够接收一组文档的初始值，并随着文档的更新而不断更新。除了 Kafka key compaction 模型(每个客户端都对每个文档更新感兴趣)之外，Firestore 还允许[更新过滤](https://cloud.google.com/firestore/docs/query-data/queries)仅接收您感兴趣的条目的更新。

使用 Firestore 可以直接实现状态复制模式。当客户机连接时，它们可以为它们感兴趣的一组键创建一个监听器。侦听器将开始向客户端传递所有这些键的当前值，然后发送客户端连接时发生的对键集的任何更新。当客户端在会话中首次连接时，它们将收到完整的源数据集。

Firestore 也不要求所有客户端都使用监听器模式。如果一个特定的用例可以由一个或另一个更好地服务，那么您可以将标准文档查询与侦听器混合使用，这没有问题。

**实施对比**

下面将比较 Kafka key compaction 和 Firestore Listeners 的用户向应用程序的所有前端传播某种配置的情况。假设我们需要跟踪定义对象配置的一组属性，比如用户表达的一组偏好。这也可能是内容管理系统中文档的一组修订，或者是仓库库存中项目的数量和位置。我们将此建模为从用户标识符到 UserConfig 协议缓冲区(或 protobuf)消息的映射。为了提高效率，我们希望在前端应用程序内存中保存这个地图的最新副本。以下接口将使用两个系统实现:

```
interface ConfigCache extends AutoCloseable {
  // Returns the configuration if it exists, or null if it does not.
  // Does not block on I/O.
  @Nullable UserConfig get(String user_id);
};
```

**卡夫卡**

在 Kafka 中，这种模式可以使用一个键压缩的`user_configs`主题来实现，这个主题代表所有用户配置的真实来源，而`user_id`作为它们的消息键。当一个配置被更新时，一个新的消息将被写入主题，UserConfig protobuf 消息被序列化为有效负载字段。此外，要删除键，需要发布一个带有特殊 tombstone 头的新消息，因为 Kafka 主题总是为每个键保留至少一个记录。

**Firestore**

当使用 Firestore 时，可以使用一个监听器来实现这种模式。用户配置将作为文档插入到`user_configs`集合中，文档 id 为`user_id`。当一个配置被更新时，一个文档将被写入集合，其中 UserConfig protobuf 消息被序列化为一个`payload` Blob 字段。

**可扩展性**

Kafka 以其水平可伸缩性和可配置性而闻名。Cloud Firestore 自动随读取流量扩展，并提供区域和多区域[高可用性](https://cloud.google.com/firestore/sla)实例。然而，它有一些[限制和配额](https://cloud.google.com/firestore/quotas),它本身可能不适合 Kafka key compaction 的所有现有用户:

*   每个数据库 10，000 次文档写入/ 10 MiB/s 写入速率:这可以通过将大量有效负载转移到带外(例如在 GCS 上)并只将 URL 放入数据库来解决。
*   每个文档每秒 1 次写入(持续一段时间):这只会影响希望快速更改单个键的用户。应该注意的是，异步复制的每个键的高写入速率意味着本地缓存将很少与事实的来源一致，不管用于实现该模式的系统是什么。
*   100 万个并发连接:这远远超过了大多数后端应用程序的预期。
*   1mb 最大文档大小:这与 Kafka 的默认值相同，但对于 Firestore 不能增加。较小的文档在发送给侦听器时会导致较低的延迟。

**成本**

上述配置系统的成本分析假设有 10，000 条记录，每条记录每天更改一次。假设每个配置的大小为 1kb，总配置大小约为 10 MiB。它还假设有 1，000 台服务器在监听这个配置中的变化，每台服务器每天重新启动一次以获取新代码。文档阅读总量将为:

*   每台服务器重启时每天 1，000 * 10，000 次
*   全天平均 1，000 * 10，000

这相当于每天阅读 20，000，000 份文件。使用[北弗吉尼亚地区定价](https://cloud.google.com/firestore/pricing)，每天的阅读成本为 6.60 美元。存储成本可以忽略不计。

应该注意的是，Firestore 监听器允许对感兴趣的子集的查询进行过滤，这相应地降低了处理和访问的成本。例如，如果某个服务器只需要为每个当前连接的用户加载配置，您可以使用以下命令监听单个用户配置的更改:

```
db.collection("user_configs").document("some_user_id")
    .addSnapshotListener(…);
```

或者，您可以筛选配置的子集，其中配置应该提供更低的延迟，例如:

```
db.collection("user_configs")
    .whereEqualTo("is_vip_user", true)
    .addSnapshotListener(…);
```

**结论**

Google Cloud Firestore 可以用来实现事件源模式，类似于 Kafka。Firestore 没有采用一种高效的方式来使用日志，而是提供了一种更高级别的 API 来有效地执行相同的操作:它允许客户端检索一组文档的初始值并监听其中的变化。一些额外的优点是，它提供了检索任意键子集或直接访问键状态的灵活性。与 Kafka 相比，Firestore 的缺点主要是它的一些可扩展性限制。参见[云 Firestore 文档](https://cloud.google.com/firestore/docs/quickstarts)了解更多关于如何创建 Firestore 数据库的详细信息。