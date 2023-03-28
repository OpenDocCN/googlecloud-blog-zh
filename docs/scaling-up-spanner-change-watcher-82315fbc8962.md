# 扩大扳手更换观察器

> 原文：<https://medium.com/google-cloud/scaling-up-spanner-change-watcher-82315fbc8962?source=collection_archive---------1----------------------->

Google Cloud Spanner Change Watcher 是 Google Cloud Spanner 的一个[开源库](https://github.com/cloudspannerecosystem/spanner-change-watcher)，用于查看和发布来自云 Spanner 数据库的更改。该库支持不同的标准实现来观察表的变化。本文中的示例需要 1.1.0 或更高版本的扳手变化观察器。

![](img/5ea3e6661905a239ecd25ee94bea5a27.png)

使用 Spanner Change Watcher 时，最重要的一个方面是如何使监视表变化的查询尽可能高效。为提交时间戳列添加辅助索引是最常用的选项之一。这带来的缺点是，由于索引值将单调增加，它可能[成为写入热点](https://cloud.google.com/spanner/docs/schema-design#primary-key-prevent-hotspots)。向二级索引添加一个[分片列是减轻这一缺点的常用策略。](https://cloud.google.com/spanner/docs/schema-design#fix_hash_the_key)

然而，将 shard 列添加到已经被一个或多个应用程序使用的现有数据库可能是一个挑战，因为它可能还需要修改现有的应用程序。本文将向您展示如何:

1.  无需修改任何现有应用程序，即可向现有数据库添加分段列。
2.  添加一个辅助索引，并使用它来有效地查询表中的数据修改。
3.  添加一个可选的 Processed (boolean)列，指示行是否已被处理，并在处理完行后使用该列从辅助索引中删除行。

# 添加计算碎片列

此示例假设您的数据库尚未包含逻辑碎片列。如果是这样，您也可以使用该列，而不是添加一个计算碎片列。

一个好的 shard 列应该包含一个值，确保写操作在多个 Cloud Spanner 服务器上平均分布。shard 列中不同值的数量应该至少与您的 Cloud Spanner 实例中的[服务器一样多。**碎片列中有太多**不同的值**不推荐**，因为轮询更改的最有效方式是在轮询查询中包含所有可能的碎片值，或者为每个碎片设置一个单独的监视器。](https://cloud.google.com/spanner/docs/instances#node_count)

因此，哈希值的模对于 shard 列来说是一个很好的值。云扳手支持[多种哈希函数](https://cloud.google.com/spanner/docs/hash_functions)。这个例子使用了 [FARM_FINGERPRINT](https://cloud.google.com/spanner/docs/hash_functions#farm_fingerprint) 函数，但是任何可用的散列函数都可以。然后，通过取散列的模 19，不同值的数量减少到 37(因此碎片值的范围是[-18，18])。

假设我们的数据库已经包含下表:

基础歌手表

以下计算碎片列可以添加到表定义中，而无需更改任何现有的应用程序:

基于全名添加计算的 ShardId 列

Cloud Spanner 将为表中的每一行自动填充这个 shard 列，而无需更改任何现有的突变或现有应用程序生成的 DML 语句。这个示例通过散列两个数据列来计算碎片，但是您可以使用任何列组合来计算碎片，只要输入对于大多数行来说足够不同以生成随机散列值。

# 添加(空筛选的)二级索引

我们现在可以为 shard 和 commit 时间戳列添加一个辅助索引，而不必担心创建一个[热点](https://cloud.google.com/spanner/docs/schema-design#primary-key-prevent-hotspots)。我们将创建二级索引作为一个经过空值过滤的索引。这将使我们能够自动从二级索引中删除不再需要的行，这可以为只接收或主要接收插入而不接收或很少接收更新的表保持较小的二级索引。

为分片和提交时间戳创建空过滤索引

如果给定行的`ShardId`或`LastModified`为空，则该行不会包含在二级索引中。在这个例子中，`ShardId`将总是非空的，因为它是歌手全名的计算哈希值。然而，我们可以选择让一个后台任务或[云函数](https://cloud.google.com/spanner/docs/use-cloud-functions)将最近没有更新的行的`LastModified`设置为 null。这将自动从辅助索引中删除这些行，并保持索引较小。

这对于大多数突变都是插入的表尤其有效。数据表将继续增长，而索引可以保持较小，方法是删除超过 X 时间前插入的条目，而无需删除实际数据。

# 使用扳手更换观察器中的索引

我们的歌手表和索引的定义如下所示:

完整的表和索引定义

我们现在将在 [Spanner Change Watcher](https://github.com/cloudspannerecosystem/spanner-change-watcher) 中设置一个表观察器，它将监视这个表的变化，并强制观察器使用我们已经创建的二级索引。这是通过在观察器上设置两个值来实现的:

1.  设置一个`ShardProvider`:shard provider 可以用来只观察表的特定部分(shard)。在这种情况下，我们将使用`[F](https://github.com/cloudspannerecosystem/spanner-change-watcher/blob/master/google-cloud-spanner-change-watcher/src/main/java/com/google/cloud/spanner/watcher/NotNullShardProvider.java)ixedShardProvider`,它将向每个投票查询添加一个`WHERE ShardId IS NOT NULL AND ShardId IN (...)`子句。在这个例子中，我们将设置分片提供者来监视所有可能的分片值，因此值列表将等于-18 和 18 之间的所有整数。创建一个包含所有可能的 shard 值的 ShardProvider 似乎很奇怪，但这实际上有助于为 Cloud Spanner 生成一个性能良好的轮询查询。
2.  设置表提示:表提示可用于强制 Cloud Spanner 使用特定的二级索引进行查询。

扳手更换观察器的示例设置

上面的表监视器将使用`ShardId`和`LastModified`上的二级索引来轮询`Singers`表的变化。轮询查询将包括以下 WHERE 子句:

`WHERE ShardId IS NOT NULL AND ShardId IN (-18, -17, ..., 17, 18) AND LastModified > @lastCommitTimestamp`。

这个`WHERE`子句意味着`ShardId`和`LastModified`都不能为空，这就是为什么我们也知道可以使用我们的空过滤索引。

# 从二级索引中删除条目

通过定期从索引中删除不再需要的条目，我们可以保持表监视器使用的辅助索引较小。这是通过将超过一定时间没有更新的行的值`LastModified`设置为 null 来实现的。重要的是，我们只有在知道这些行不会被其他进程同时更新时才这样做，因为这可能导致这些行的事务性更新不会被更改监视器拾取。

下面的示例脚本将把过去 24 小时内没有更新的所有行的`LastModified`设置为 null。这个脚本应该作为[分区 DML](https://cloud.google.com/spanner/docs/dml-partitioned) 执行，以确保它不会超出 Cloud Spanner 的事务限制。

# 使用经过处理的列

上面的例子假设对于一段时间没有更新的行，可以将`LastModified`设置为 null。如果希望保留旧行的`LastModified`值，可以引入一个额外的`Processed`列来指示更新是否已经被处理并且不再需要被表观察器考虑。该`Processed`列可用于在行被处理后自动将`ShardId`列设置为空。由于空过滤索引将只包含所有包含的列都不为空的条目，所以我们将列设置为空并不重要。这两种情况都将导致从辅助索引中删除该行。

下面的表和索引定义将创建一个表，该表为尚未标记为已处理的行自动计算一个`ShardId`。因此，索引将只包含未标记为已处理的行。

具有已处理列的表定义

上述表格和索引定义可与`FixedShardProvider`结合使用。然而，它确实需要修改现有的**更新语句，以将** `**Processed**` **列更新为假**(或空)。不需要修改现有的 insert 语句，因为这些语句会将 null 插入到`Processed`列中。`CASE WHEN Processed`子句将 null 和 false 都视为 FALSE 值，并将`ShardId`设置为非 null。

## 设置变更观察者

该表的变更监视器的设置方式与不包含`Processed`列的版本完全相同。

表监视器不会考虑任何`ShardId`为空的行，这样做很有效，因为它可以使用只包含相关行的二级索引。

## 从二级索引中删除条目

从二级索引中删除不再需要的条目与前面的示例略有不同。现在通过将`Processed`列设置为真来删除条目。这将自动将`ShardId`列更新为空。`LastModified`列将保持不变。

标记已处理行的查询

# 基准应用程序

Cloud Spanner Change Watcher 示例目录还包括一个小型基准测试应用程序，可用于测试和基准测试不同的配置。基准应用程序的默认配置**是为必须处理高写吞吐量的变更观察者推荐的基本配置**，与本文中的示例相同。这种配置能够在单个节点云 Spanner 实例上每张表每秒处理大约 3，000–5，000 个突变。

您可以更改基准测试应用程序的输入参数，在您自己的 Cloud Spanner 实例上尝试不同的配置。[关于基准应用程序的更多信息可以在本文](/@knutolavloite/benchmark-spanner-change-watcher-e5b6cc2ac618)中找到。

# 结论

在接收大量写操作的表的提交时间戳列上创建二级索引，可以创建一个[热点](https://cloud.google.com/spanner/docs/schema-design#primary-key-prevent-hotspots)。添加一个 shard 列作为索引的第一列可以缓解这个问题。如果您的数据库不包含逻辑碎片列，您可以添加一个作为计算列，用于计算表中一个或多个数据列的哈希值。计算列将由 Cloud Spanner 自动填充，这意味着您不需要修改任何现有的应用程序来使用这样的列。

在 shard 和 commit timestamp 列上创建一个二级索引作为空过滤索引，还可以通过将索引中的一个值设置为 null 来删除二级索引中不再需要的旧条目。上述示例显示了如何通过将提交时间戳值设置为 null 来实现这一点。

但是，如果您希望保留旧条目的提交时间戳值，那么您也可以引入一个单独的`Processed`列来指示该行是否已被处理。该列可用于自动将计算碎片列设置为 null，这也会自动从辅助索引中删除该行。