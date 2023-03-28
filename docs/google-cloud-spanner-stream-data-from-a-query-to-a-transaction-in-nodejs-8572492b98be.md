# Google Cloud Spanner:从查询到 NodeJS 中的事务的数据流

> 原文：<https://medium.com/google-cloud/google-cloud-spanner-stream-data-from-a-query-to-a-transaction-in-nodejs-8572492b98be?source=collection_archive---------4----------------------->

[Google Cloud Spanner](https://cloud.google.com/spanner) 是一个全面管理、可扩展的关系数据库服务，用于区域和全球应用数据。它是第一个可扩展的、企业级的、全球分布的、高度一致的数据库服务，专为云构建，将关系数据库结构的优势与非关系水平扩展相结合。

![](img/5ea3e6661905a239ecd25ee94bea5a27.png)

Cloud Spanner 中的查询可以使用[流 RPC](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1#google.spanner.v1.Spanner.ExecuteStreamingSql) 来执行。 [NodeJS 客户端库](https://github.com/googleapis/nodejs-spanner)使用这个流 API 以标准 NodeJS 流的形式返回查询结果。这些流可以通过管道与其他流一起将数据写入其他输出，如果写流比查询流慢，NodeJS 会自动对读流施加反压力。这篇文章展示了如何在 Cloud Spanner 上将查询结果流式传输到读/写事务中，以便将(一些)查询数据写回 Spanner。

[Database#runTransaction](https://googleapis.dev/nodejs/spanner/latest/Database.html#runTransaction) 函数应用于在事务中向 Cloud Spanner 写入数据。然而，这不是一个流，NodeJS 只会在读流通过管道进入另一个流时对其施加反压力。因此，我们需要实现自己的自定义流实现，将数据写入 Spanner。参见 [NodeJS 文档](https://nodejs.org/docs/latest/api/stream.html#stream_implementing_a_writable_stream)了解更多关于如何实现定制写流的背景信息。

# 步骤 1:每个事务写一行

我们将从一个简单的实现开始，每个事务只写一行。为此，我们需要创建一个简单的 Writable 实现来覆盖 [Writable。_ 编写](https://nodejs.org/docs/latest/api/stream.html#stream_writable_write_chunk_encoding_callback_1)方法。

可写流，每个事务写一行

上面的代码示例将选择 Singers 表中的所有行，然后将所有这些行写回到同一个表中，有效地复制了表中的所有行。每一行都在单独的事务中写入。

NodeJS 将对读流施加反压力，并保持较低的内存消耗，因为写流比读流慢得多。我们不需要为背压的发生实现任何自定义逻辑。

确切的内存消耗将取决于正在读取的表的大小以及 Cloud Spanner 如何对查询结果进行分区。查询结果由一个[PartialResultSet](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1#google.spanner.v1.PartialResultSet)流组成，内存使用量将至少与一个 partial resultset 的大小一样大。

# 步骤 2:每个事务写多行

上面的例子是可行的，但是效率非常低。最好是将更多的行批处理在一起，并在一个事务中写入这些行，而不是启动一个单独的事务。这可以通过实现可写的[来实现。_writev](https://nodejs.org/docs/latest/api/stream.html#stream_writable_writev_chunks_callback) 方法而不是可写的。_ 写。

此外，我们可以在传递给流构造函数的选项中设置一个 [highWaterMark](https://nodejs.org/docs/latest/api/stream.html#stream_new_stream_writable_options) 。对于对象流，highWaterMark 的默认值是 16。这意味着在创建一个事务并将它们写在一起之前，我们的流最多只能缓冲 16 行。根据行的大小和可用内存，该值可以增加到更高的值。较高的值将确保较高的吞吐量，但会增加内存消耗。

此示例使用 200 行的高水位线。这可以进一步增加，但必须保持在 Cloud Spanner 的[交易限额](https://cloud.google.com/spanner/quotas#limits_for_creating_reading_updating_and_deleting_data)内。一个事务最多可以写入 20，000 个变异，其中一个变异大约等于每个变异中的行数*列数。

可写流，每个事务最多写入 200 行

# 结论

来自 Cloud Spanner 的查询结果可以作为 NodeJS 流返回，并通过管道传输到任何其他 NodeJS 可写流中。上面的例子已经表明:

1.  如何创建自定义流以将数据写入云扳手事务。
2.  我们如何使用 NodeJS 流的内部缓冲和背压机制来提高从查询到事务的数据流的执行速度，同时保持对内存使用的控制。