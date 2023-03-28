# 云世界中数据库管理员的发展

> 原文：<https://medium.com/google-cloud/evolution-of-a-dba-in-the-cloud-world-ecee5fffadde?source=collection_archive---------3----------------------->

我在技术领域的第二份工作是数据库管理员。我真的很喜欢这份工作，因为我不仅要处理表模式、索引(我仍然想念 [INGRES](https://en.wikipedia.org/wiki/Ingres_(database)) ，因为我要选择正确的索引类型)、查询优化、访问路径、备份&恢复过程、安全性、ETL 作业等等。我还花了几个小时查看队列深度、iops 和延迟，设计 SAN 查看 LUN 并配置适当的 RAID 级别。这可能是我唯一一次能够想出一个有意义的颜色编码方案！

然后，当它不仅仅是选择哪种类型的关系数据库，而是我需要哪种非 sql 数据库时，它变得更加有趣(这弥补了我甚至不必考虑我需要什么类型的索引的不足)😀) .回到最近的时代和“云”，现在担心 LUN 和条带化磁盘不是一件事，至少在我最近工作的云上不是。

我过去作为 DBA 做的许多任务已经被抽象掉了，因为现在这是 GCP 的问题，但其他关注点已经介入，填补了这一低水平的无差别重担。

关于如何在云中管理数据库，有两种选择，我将在这里重点介绍 GCP(假设你们都参与了这篇文章)，要么在计算引擎(GCE)上部署和管理自己的数据库，要么使用完全托管的数据库服务之一。

## 在 GCE 上部署和管理数据库

因此，我知道我在这里只是略述皮毛，但是我只是想让您了解一下作为 DBA 在部署到 GCE 时需要考虑的任务。其中一些任务与您在内部完成的任务相同，但其他任务实际上相当容易完成。

如果直接部署到 GCE 上，那么您需要深入了解将要像在本地一样使用的数据库软件的操作方面。您需要像通常一样将数据和日志从引导磁盘分离到不同的磁盘上，这种方法实际上并没有改变。备份和恢复过程使用云存储(GCS)近线从持久磁盘(PD)移动数据库备份文件。持久磁盘是耐用的存储设备，其功能类似于台式机或服务器中的物理磁盘。计算引擎管理这些设备背后的硬件，以确保数据冗余并为您优化性能。

如果您想要复制分层存储系统，那么可以实施一个过程，允许您将备份从 PD 移动到标准 GCS，然后移动到近线 GCS。监控和日志记录您可以使用 stackdriver，因为它有一些现成的[代理](https://cloud.google.com/monitoring/agent/plugins/)来监控一些常见的数据库。省去您设置额外的监控基础设施

需要 RAM(你当然需要！)选择一个能提供您所需 RAM 容量的实例类型，或者如果没有合适的实例类型，就定制一个。

数据库(在这篇文章中，我们忽略内存数据库！)，磁盘 I/O 和吞吐量一直是您需要关注的一个关键方面。你仍然需要关注这些参数，但这是你真正需要改变思维的地方。现在，您不必担心 DAS、NAS 或 San、光纤通道和 RAID 级别。面向数据中心专业人员的 GCP 文档有一个[精美的章节](https://cloud.google.com/docs/compare/data-centers/storage#storage_area_network_san)帮助您理解从 SAN 和归档存储到 PD 和 GCS 的思维转换。

注意:你确实使用了本地 SSD，但是它们被绑定到底层主机，所以只适合暂存盘和可以快速重建的盘。

现在，您最终需要关心的是 PD 的类型、PD 的大小，以及如果您使用 SSD PD，vCPUs 的数量。

文档中关于比较[块存储性能](https://cloud.google.com/compute/docs/disks/performance#type_comparison)的表格是一个很好的起点，或者如果您想要比[表格](https://cloud.google.com/compute/docs/disks/performance#comparing_persistent_disk_to_a_physical_hard_drive)更进一步，将 7200 rpm SATA 与标准 PD 或 SSD PD 进行比较有助于了解您应该如何考虑块存储性能，并使用它来适当确定 PD 的大小。

关于[磁盘优化](https://cloud.google.com/compute/docs/disks/performance)的整个页面值得一读再读，但这里有一些我个人认为永远值得记住的要点:

*   标准持久磁盘性能可线性扩展至卷性能极限。您的实例的 vCPU 数量不会限制**标准**持久磁盘的性能。
*   SSD 持久磁盘性能线性扩展，直到达到卷的极限或每个计算引擎实例的极限。一般来说，具有更多 vCPUs 的实例可以实现更高的吞吐量和 IOPS 限制。
*   每个持久磁盘写入操作都会增加虚拟机实例的累积网络出口流量。
*   您仍然需要考虑队列深度和并行性，并进行相应的优化，

一个很好的例子是如何将所有这些整合在一起，并考虑在 GCE 上管理和优化数据库，这是 GCP 网站上关于 GCP 上的 Microsoft SQL Server 的最佳实践的文档。

因此，现在一名 DBA 在更高的抽象层次上优化 I/O 和吞吐量，而 GCP 则负责这种无差别的繁重工作(尽管我确实没有意识到这最终并没有最好地利用我的时间！).基本上，一旦你明白你需要关注什么，较低层次的基础设施问题就变得容易多了。

## GCP 全面管理的数据库

有了完全托管的数据库，您就不再关心 PD 大小和优化块大小等细节，从而最大限度地提高性能。现在，您将重点放在为每项服务提供的个别指导上。

我建议，如果你是 GCP 的新用户，并且已经做出了完全明智的决定，那就从[这篇关于 GCP](https://cloud.google.com/solutions/data-lifecycle-cloud-platform) 的[数据生命周期的](https://cloud.google.com/solutions/data-lifecycle-cloud-platform)文章开始吧(需要添加扳手提示提示！) .一旦你读了，然后阅读概念指南，并着手快速启动。

下面我列举了几个 GCP 托管数据库服务的例子，以及一些优化使用它们的方法。

**BigTable**

*   阅读[了解云大表性能](https://cloud.google.com/bigtable/docs/performance)
*   存储量越大，性能越好(因此> 1 TB)
*   选择一个[块存储类型](https://cloud.google.com/bigtable/docs/choosing-ssd-hdd)

**大查询**

*   了解[分区表](https://cloud.google.com/bigquery/docs/partitioned-tables)的使用
*   调整您的查询
*   使用[查询计划](https://cloud.google.com/bigquery/query-plan-explanation)来帮助提高性能
*   使用[启动清单](https://cloud.google.com/bigquery/launch-checklist)

**数据存储**

*   阅读[最佳实践](https://cloud.google.com/datastore/docs/best-practices)
*   阅读[平衡强大的&与谷歌云数据存储的最终一致性](https://cloud.google.com/datastore/docs/articles/balancing-strong-and-eventual-consistency-with-google-cloud-datastore/)

传统数据库管理员通常会执行的任务基本上已经上移了。这意味着有更多的时间参与其他事情。

存储和查询数据只是冰山一角，但是将数据放入数据存储以及云和 ETL 工作都是 DBA 的一部分。我没有忘记诚实！

## 抽取、转换、加载至目的端（extract-transform-load 的缩写）

集成 Hadoop 及其生态系统是解决各种数据问题的常用途径。也许我应该把它作为一个典型的 DBA 现在所做的事情的一部分(但是我想我是从我当 DBA 时所拥有的工具开始的，而 Hadoop 不是其中之一)。

我第一次涉足 Hadoop 及其生态系统实际上是在 AWS 的 EMR 上，因此我错过了桌子下和橱柜中的大量商用硬件，这是在内部运行 Hadoop 基础架构的一部分。我有一个相对无痛的介绍。看看 GCP 的 [Dataproc](https://cloud.google.com/dataproc/docs/concepts/overview) 这真的是 Hadoop 类型的任务应该运行的方式🙂)

在 Hadoop 类型的工作负载(是的，我想我真的是指 map/reduce)之后不久，寻找解决方案来帮助使 [lambda 架构](https://en.wikipedia.org/wiki/Lambda_architecture)工作而不会让你眼珠子掉出来将是下一个理解，因此 [Apache beam](https://beam.apache.org/) 可能是下一步，或者如果你从未接触过 Hadoop，实际上是第一步。这是下一代，所以还是从那里开始吧。GCP 上的[数据流的完全管理版本。](https://cloud.google.com/dataflow/)

到现在为止，你可能会想，继续往上爬，他们不是已经把 DBA 改名为数据工程师了吗？你知道你是对的(如果你不完全确定什么是数据工程师，我喜欢这个[作为我迄今为止读过的最好的描述)。](https://medium.freecodecamp.com/the-rise-of-the-data-engineer-91be18f1e603)

数据工程师是一个进化的数据库管理员，拥有更酷的工具来帮助他们完成工作。我也见过随机加入 ML 的 mashups，但我觉得它更适合数据科学领域。老实说，我不会对你的底线进行哲学讨论，职称只是给你一个模糊的概念，告诉你某人在日常工作中主要做什么，而且界限很模糊。