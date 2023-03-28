# 构建和使用谷歌云存储对象的清单

> 原文：<https://medium.com/google-cloud/building-and-using-an-inventory-of-your-google-cloud-storage-objects-in-bigquery-1afae4333351?source=collection_archive---------0----------------------->

# 放弃

我是一名谷歌人，具体在谷歌云工作。这里陈述的所有观点都是我自己的，而不是谷歌公司的。

# 介绍

听到这个可能会让您感到惊讶，但是许多对象存储客户并不真正知道他们的存储桶中有什么数据。就像您在搜索桌面文件系统的偏远角落时一样，当他们开始寻找时，他们会发现令他们惊讶的东西。在对象存储的情况下，这在最好的情况下会导致存储费用的浪费，在最坏的情况下会导致敏感数据的泄露。很多时候，客户知道他们有这个问题，但他们对自己的数据了解不够，甚至没有开始清理。

如果这听起来像是你可能遇到的问题，那么有一个好消息。通过建立一个可以用 BigQuery 这样强大的数据库查询的对象清单，你可以很容易地[知道](https://www.youtube.com/watch?v=c71nqMqmiaw)你得到了什么，并做出明智的决定。*你不需要写任何代码就可以开始。*

# 如何做到这一点

## 拥有谷歌云平台账号和存储对象

大多数想走这条路的人已经有了一个 GCP 账户，但是如果这是云平台评估的一部分，或者你正在考虑使用 BigQuery 来清点另一个对象存储产品，那么[很容易就能从 GCP](https://cloud.google.com/start/) 开始。至于你可能期望的费用，[存储有持续的月费用](https://cloud.google.com/storage/pricing)取决于你存储了多少。它也有运营费用，但在这种情况下，这些费用可以忽略不计。BigQuery 也有持续的每月存储成本，并且查询是按使用付费的。

对于大多数用户来说，与对象存储成本相比，BigQuery 的存储和查询成本相对较低。如果您只是尝试一下，那么只需要几个对象——只需在 UI 中上传几张照片。

## 用我为此编写的工具克隆 repo 并运行它

我的戏剧化，希望能节省你的时间和精力。

为了节省您大量的时间和精力，**我继续编写了** [**命令行实用程序，它列出对象并将它们流式传输到 BigQuery**](https://github.com/domZippilli/gcs-inventory-loader) 。在一个相当大的(~32 vCPU)虚拟机上，它应该每 150 秒运行大约 1，000，000 个对象。我建议从 GCE 虚拟机中这样做，以获得最佳吞吐量并避免任何出口费用，但它也可以在家中或内部工作。

该实用程序是为 **Python 3.6** 编写的。操作原理非常简单。一次最多列出两个存储桶，并且存储桶列表的每一页(大约 1，000 个对象)都被分派到池中的一个单独的线程。这使得该实用程序可以非常快速地获取大量对象信息，然后将其传输给 BigQuery。

默认情况下，该实用程序将收集指定项目中所有存储桶的清单。如果您的项目中有一个 *lot* 对象，您可以提供特定的存储桶，以便按存储桶“分割”列表工作。您还可以使用这个特性来列出多个项目的存储桶。

如果您需要分割一个非常大的桶的列表，您还可以提供一个前缀来过滤列表。

按照 **README.md** 中的步骤安装和配置该工具，对于大多数用户来说，不需要任何选项就可以运行它:

```
gcs_inventory load
```

您应该得到输出，通知您对象计数的进度。因为它事先不知道你有多少个对象，所以它不会给出进度指示器，尽管你可以通过检查`storage/object_count` [Stackdriver 度量](https://cloud.google.com/monitoring/api/metrics_gcp#gcp-storage)来了解你有多少个对象。

## 最后，打开 BigQuery，尽情享受吧

Arvind，是时候用 SQL 从我们的数据中收集见解了！

一旦实用程序运行完毕，您就可以开始查询数据了(从技术上讲，您可以在它运行完毕之前进行查询，但是数据将是不完整的)。这里有一些 BigQuery SQL 将帮助您找到关于 GCS 对象的有趣内容。至少对我来说，发现有趣的东西总是很有趣。

**我最大的 10 个对象是什么，以 GB 为单位？**该查询将向您展示。

```
SELECT
  name,
  ROUND(size / 1000 / 1000 / 1000, 1) as sizeGB,
  storageClass
FROM `project.dataset.table`
ORDER BY size DESC
LIMIT 10
```

从这里，我可以看到我的开发项目中有一堆 2GB 的文件:

```
name       sizeGB storageClass
<redacted> 2.2    NEARLINE
<redacted> 2.2    NEARLINE
<redacted> 2.1    NEARLINE
<redacted> 2.1    NEARLINE
<redacted> 2.1    NEARLINE
[...]
```

也许这些物品不再有用了？它们占了很大空间，现在我知道了，所以我可以调查一下。

**我有重复的对象吗？**使用每个对象的 MD5 或 CRC32C 校验和来检测重复数据。在这个例子中，我将使用 CRC32C，因为所有对象都有它(组合对象没有 MD5)。

```
SELECT
  crc32c,
  size,
  COUNT(crc32c) AS copies
FROM `project.dataset.table`
GROUP BY crc32c, size
ORDER BY copies DESC
```

由此，我可以看到我有一堆具有相同大小和校验和的对象——这些几乎肯定是重复的。

```
crc32c   size     copies
IlUZiA== 51130562 12
j0uDyA== 51130562 12
OVgmZA== 51130562 12
i2S8Ww== 1073741792 12
i2pspw== 51130562 12
RnLi+Q== 51130562 10
[...]
```

现在，我将使用这个查询和我的库存执行一个内部连接，以生成一个重复对象的“命中列表”:

```
SELECT
  object.name,
  object.crc32c,
  object.size
FROM
  `project.dataset.table` AS object
INNER JOIN (
  SELECT
    crc32c,
    COUNT(crc32c) AS copies
  FROM
    `project.dataset.table`
  GROUP BY crc32c
  ORDER BY copies DESC
) AS checksums
ON
  object.crc32c = checksums.crc32c AND checksums.copies > 1
ORDER BY object.size DESC
```

现在，我有了一组可以进一步研究并考虑进行重复数据消除的对象:

```
name       crc32c   size
<redacted> z/SBzA== 2150214565
<redacted> z/SBzA== 2150214565
<redacted> L4Z8vg== 2148600803
<redacted> L4Z8vg== 2148600803
<redacted> pb4tWw== 2147908484
<redacted> pb4tWw== 2147908484
<redacted> Mi8Jnw== 2145435876
<redacted> Mi8Jnw== 2145435876
<redacted> rs+BzQ== 2142987458
<redacted> rs+BzQ== 2142987458
[...]
```

**我的数据是否在不该在的地方？**我可以使用对象元数据来查找可能不应该在公共访问桶中的文件:

```
SELECT 
  name, 
  bucket 
FROM `project.dataset.table` 
WHERE SUBSTR(name, LENGTH(name) - 2) IN (".db") OR
      SUBSTR(name, LENGTH(name) - 3) IN (".csv", ".txt") OR
      SUBSTR(name, LENGTH(name) - 4) IN (".json") AND
      bucket = "very-public-bucket"
```

一个恶意的演员可以很容易地回避这一点，但这样的定期检查可以捕捉诚实的错误…许多大问题都是从这些开始的。

# 下一步是什么

请继续关注更多的文章，这些文章描述了您可以利用这些信息和类似的数据源来更好地管理 GCS 中的数据。

此外，元数据加载器的一些改进也即将出现。第一个是包括 ACL 和 IAM 绑定，但是我仍然处于“可行性”阶段。

# 已知问题

一个已知的问题是，如果您正在开发而不仅仅是使用这个实用程序，您更有可能遇到这个问题，即[如果您删除并重新创建一个表，并且在那之后不久就开始向其中传输数据，一些记录可能不会进入新表](https://github.com/googleapis/java-bigquery/issues/15#issuecomment-569300832)。我不确定一个被删除的表会有多长时间的影响，但是每次运行这个函数时，您可能希望将新的清单加载到一个新的表中。

# 警告

1.  许可证文件中已经说明了这一点，但是`gcs-inventory-loader` repo 中的所有代码都是按原样提供的，没有任何保证。**谷歌**不支持该实用程序。在运行代码之前，请确保您彻底理解了代码，这样您就知道它不会给您带来问题。
2.  这样创建的库存**不是快照**。差别很微妙，但很重要。“快照”指的是在特定时间点一致的 GCS 元数据清单。列表 API 不是这样工作的；每个页面都是实时元数据的一部分，因此当您浏览页面时，其余的元数据可能会发生变化。对象可能会被您遗漏或删除，并显示在您的清单中。产生一致的快照在技术上是可能的，但是方法是完全不同的，并且现在需要大量的基础工作。那是改天的话题。

![](img/ca09775d510cfc354bae19406b5eb6b9.png)

在谷歌数据中心里。我想我看到你的东西了！