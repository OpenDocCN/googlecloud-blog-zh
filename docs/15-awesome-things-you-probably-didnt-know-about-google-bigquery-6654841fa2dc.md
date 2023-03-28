# 关于谷歌大查询，你可能不知道的 15 件大事

> 原文：<https://medium.com/google-cloud/15-awesome-things-you-probably-didnt-know-about-google-bigquery-6654841fa2dc?source=collection_archive---------0----------------------->

Google BigQuery 诞生于 2012 年的 Dremel，是一个非常独特的分析数据仓库服务。BigQuery 通常被描述为无服务器、无操作、无缝可伸缩和完全可管理的。因为 BigQuery 确实没有对等的东西，所以有必要提一下是什么让 BigQuery 如此神奇的一些不太明显的方面！

1.  **加密**

BigQuery ( [和一般的谷歌云](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwiZ8Jy84uvPAhXhilQKHQzhAVQQFggkMAI&url=https%3A%2F%2Fcloud.google.com%2Fsecurity%2Fencryption-at-rest%2F&usg=AFQjCNHcn98lloEZ-CwUqaD33wNMJCagHQ&sig2=rTtkON_LjpTR7iFefIRgdw&bvm=bv.136499718,d.cGw))非常重视安全性。例如，默认情况下，BigQuery 会加密所有静态和传输中的数据。我们认为加密对我们所有客户的安全至关重要。

**典型的分析数据库因加密而遭受 30%-50%的性能打击**。在进行任何性能比较时，记住这一点很重要。

**2。高效并发**

与传统的基于虚拟机的分析数据库不同，BigQuery [依赖于谷歌规模的子服务，如 Dremel 和 Colossus](https://cloud.google.com/blog/big-data/2016/01/bigquery-under-the-hood) 的存储、计算和[内存](https://cloud.google.com/blog/big-data/2016/08/in-memory-query-execution-in-google-bigquery)设施。所有这些都由谷歌的[千兆比特的木星网络](https://research.googleblog.com/2015/08/pulling-back-curtain-on-googles-network.html)联系在一起，该网络本质上允许每个节点以 10G 的速度与其他每个节点对话。

利用这种架构的主要好处之一是**无瓶颈扩展和并发性**。高度优化的千篇一律的性能基准测试经常会忽略混乱的现实世界，充满了不可预测的工作负载，这些工作负载会争夺公共资源，并遇到与基于虚拟机的架构相关的各种瓶颈。

请允许我向您介绍一个团队的帖子，该团队[针对各种其他技术](https://cmtoomey.github.io/speed/2016/11/04/tableaudragrace.html)评估了 BigQuery，展示了并发性对真实用例的影响。

这个故事的寓意是，并发在现实世界中很重要，BigQuery 独特的架构非常好地处理了并发。我们鼓励你自己去发现！

**3。没有分配或分类键**

BigQuery 的存储并不位于虚拟机上，而是位于一个名为 Colossus 的高性能分布式存储系统上。这使得 BigQuery 不需要任何排序或分布键！

除了额外的操作开销，要求您定义这样的键的数据库限制了您进行真正的特别 SQL 分析的能力。次优利用排序键定义的查询可能会执行次优。在考察这类数据库的性能时，重要的是测量真正不可预测的查询，而不是专门为排序关键字定义优化的查询。

**4。超高效、真正的云原生定价**

如果我告诉你启动数千台服务器，在其上安装和配置分布式大数据服务，将数据加载到数据库中，运行 SQL 查询，然后关闭整个系统，会怎么样？你会说我极其浪费。如果我告诉你，你可以在 1 秒钟内完成以上所有的事情，会怎么样？你会说我疯了。如果我告诉您，我们将向您收取相当于每秒计费的费用，会怎么样？你会说我是骗子！

这就是【BigQuery 的按需定价模型给你的。这是真正的[云原生和超高效](https://cloud.google.com/blog/big-data/2016/02/visualizing-the-mechanics-of-on-demand-pricing-in-big-data-technologies)。

这种效率对于分析工作负载尤为重要。运行 Dremel 10 年和运行 BigQuery 年的经验告诉我们，分析工作负载非常不稳定。事实上，随着需求的增长，分析工作负载变得更加不稳定(与 OLTP 不同)。

因此，**任何以 0 波动性(或标准差)模拟分析工作负载的定价比较都是极不现实的情况。可悲的是，现实世界并不是这样的。**

换句话说，BigQuery 既提供了实时无瓶颈资源分配的便利(当然是在合理的范围内)，又保证了 100%的资源利用率。你只为你消费的东西付钱。

相比之下，各种行业研究告诉我们，您很幸运能够以超过 25%的利用率运行您的分析工作负载，这意味着您支付了 4 倍于您应该支付的费用。[有些人甚至故意以 50%的利用率运行](https://www.periscopedata.com/blog/building-the-periscope-cache-with-amazon-redshift.html)，故意将他们的月账单翻倍。

当然，有很多理由用成本效率来换取成本可预测性，尤其是在大型组织中。出于这个原因，BigQuery 现在采用统一费率定价。你支付固定的月费，所有的 SQL 查询都是免费的。除了它只是 Dremel 中的一个资源配置，而不是一个实际的集群之外，您得到了一个“大查询集群”的等价物。

**5。持续自我优化存储**

2016 年 3 月，BigQuery 发布了[电容器，一种新的最先进的存储格式和系统](https://cloud.google.com/blog/big-data/2016/04/inside-capacitor-bigquerys-next-generation-columnar-storage-format)。电容器的许多有趣的方面之一是它固执己见的存储方法。后台进程不断评估客户的使用模式，并经常自动优化这些数据集以提高性能。这对于最终用户来说是完全自动化和透明的。实际上，随着时间的推移，您的查询会变得更快，因为您正在教授 BigQuery。

同样，我们永远不会要求您整理、清空或重新上传数据集到 BigQuery，这是我们的完全托管存储理念，我们希望这对我们的客户非常有吸引力。

90 天后，大概是因为我们已经用尽了优化您的数据集的能力，我们将节省下来的成本转移给我们的客户，并自动将存储价格降低 50%，而不会降低性能或耐用性。

**6。高可用性**

BigQuery 是高度现成可用的。所有客户数据在地理位置上无缝复制，我们的 SREs 管理您的查询在哪里执行。根据具体情况，您可能会在一个数据中心开始一天的工作，但几小时后又会无缝地在另一个数据中心结束工作。

创建高度可用的分析服务非常困难。它需要至少 3 倍的部署(和成本)和非常重要的技术复杂性，如[这个](https://aws.amazon.com/blogs/big-data/building-multi-az-or-multi-region-amazon-redshift-clusters/)解决方案所示。

**7。缓存**

BigQuery 提供免费的每用户缓存。如果您的数据没有变化，您确定性编写的 SQL 的结果将自动缓存 24 小时。下次运行相同的查询时，您的查询将免费执行，并且您的结果几乎会立即提供。

在设计定期刷新 SQL 查询的部署时，缓存是一个重要的考虑因素。如果这些刷新命中您的缓存，您不会浪费任何金钱或资源来重新计算已经完成的工作。缓存节省了金钱、时间和资源。

**8。免费批量摄取**

BigQuery 的批量摄取是将数据加载到 BigQuery 的一种很好的方式。这种方法是完全免费的，或者实际上大部分分摊到存储成本中，而不是计算。

与传统分析技术不同，批量接收不会消耗专用于分析和 SQL 的池中的计算资源。换句话说，无论您将多少数据装载到 BigQuery 中，您的查询能力都不会减少一点。

批处理摄取也完全是原子性的，这意味着作业要么同时成功，要么同时失败——没有令人讨厌的竞争条件或部分负载需要清理。

**9。使用流式 API 进行实时摄取**

BigQuery 的流 API 允许您每秒为每个表加载多达 100，000 行，以便在 BigQuery 中进行即时访问。一些客户通过在几个表之间分片来实现每秒数百万行。

流式 API 确实有自己的成本，但是，同样，传统的分析数据仓库不提供免费的批处理或流接收，因为成本负担是通过消耗付费资源来感受的，否则这些资源将专用于 SQL。

**10。无服务器、无操作、完全托管**

我以前曾[说过](https://cloud.google.com/blog/big-data/2016/08/google-bigquery-continues-to-define-what-it-means-to-be-fully-managed)当谈到完全可管理性时，BigQuery 是独一无二的——我们知道您的工作何时失败，我们的 sre 全天候待命，我们无缝地执行无停机升级和维护，并且我们永远不会要求您重启或调整您的“BigQuery 集群”。

Parsely 的 CTO 在他自己的博客中反映了这些观点。

最后，无论从哪个角度来看，BigQuery 都称得上是真正的无服务器。

**11。轻松共享数据**

想象一下，与您的同事、客户或合作伙伴共享 Pb 大小的数据集，就像您共享您的 Google 电子表格和文档一样——通过将数据保留在原位并修改访问控制列表(ACL)。

传统的云数据仓库建议旋转不同的集群，并将数据加载到这些集群中。这当然增加了复杂性、成本和运营开销。BigQuery 的方法优雅而高效，由 BigQuery 独特的无服务器架构提供。

**12。公共数据集**

BigQuery 采用了这种“简单数据共享”的概念，并将其进一步扩展。如果任何人都可以访问数据集会怎样？现在，TB 级的公共数据集可立即用于 SQL 分析，甚至无需启动虚拟机、配置数据库和将数据加载到集群中。

[空格或制表符](/@hoffa/400-000-github-repositories-1-billion-files-14-terabytes-of-code-spaces-or-tabs-7cfe0b5dd7fd#.5ijy25fhm)？ [Kubernetes 还是 swarm](/google-cloud/analyzing-github-issues-and-comments-with-bigquery-c41410d3308#.6tg9weheu) ？Python 还是围棋？微软还是苹果？Github 数据集回答了这些问题以及更多问题。其他有趣的数据集包括最新天气、[世界大事](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=7&cad=rja&uact=8&ved=0ahUKEwjr2djuzezPAhUCLmMKHa_nAncQtwIIRzAG&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3DPsp7YivWL90&usg=AFQjCNHN80KiBtj3oLCLrBpjt1sChSnRPg&sig2=FXQHi9YuqsIEbw5VL2MrLw&bvm=bv.136593572,d.cGc)、维基百科，甚至[婴儿名字](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=6&cad=rja&uact=8&ved=0ahUKEwi6_9-AzuzPAhVW0WMKHSjxCR4QFgg7MAU&url=https%3A%2F%2Fwww.reddit.com%2Fr%2Fbigquery%2Fcomments%2F585h8r%2Fthe_presidential_baby_name_bump%2F&usg=AFQjCNHn3QJMfsa6Re-HDgRsy3SB5F3aIg&sig2=7sbcyVQ3pN9J4PKiv3HZ1g&bvm=bv.136593572,d.cGc)！

或者如何确定 Github 的可靠性？谷歌 SRE 最近能够做到这一点，你也可以！

**13。Pb 级**

你敢在 1pb 大小的表上运行全表扫描吗？你敢在 Strata 的舞台上现场表演吗？BigQuery 产品经理 Chad Jennings [正是这样做的，在不到 4 分钟的时间内扫描超过 1pb，展示了令人难以置信的每秒 4tb 的吞吐量。](https://www.oreilly.com/ideas/google-bigquery-for-enterprise)

BigQuery 的一些最大的(外部)客户每天加载 1pb 的数据，并在 BigQuery 中存储超过 100 Pb 的数据。

**14。联合访问**

想在 Google 云存储中查询你的数据，而不是先加载到 BigQuery？现在，您可以使用 BigQuery 做到这一点。直接从 BigQuery 查询 Google Sheets 怎么样？这也是可能的。BigQuery 为这些用例提供了超灵活的[联邦访问模型](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwjMhqPPz-zPAhWIh1QKHaolBEUQFggcMAA&url=https%3A%2F%2Fcloud.google.com%2Fbigquery%2Ffederated-data-sources&usg=AFQjCNFQiWTIaiq9MwFQlb3O5d-6tQN1-A&sig2=icRqdLtT3YCvbOFxvTWUIw&bvm=bv.136499718,d.cGw)。

15。热情的顾客

BigQuery 的客户喜欢这项服务。BigQuery 允许他们专注于他们的业务问题和附加值，而不是搅动复杂性和运营。

BigQuery 的一些客户包括 [Spotify](https://news.spotify.com/us/2016/02/23/announcing-spotify-infrastructures-googley-future/) 、 [New York Times、可口可乐、Viant](https://cloud.google.com/blog/big-data/2016/09/bigquery-introducing-powerful-new-enterprise-data-warehousing-features) 、[摩托罗拉](https://cloud.google.com/blog/big-data/2016/07/supersize-it-how-motorola-transformed-its-data-warehousing-and-analytics-with-google-cloud-platform)、 [Kabam Games](https://youtu.be/6Nv18xmJirs?t=8m) 、 [Vine](http://stackshare.io/google-bigquery) 、[迪士尼互动](https://cloudplatform.googleblog.com/2016/08/the-dragon-days-of-summer-this-week-on-Google-Cloud-Platform.html)、[劳埃德银行](http://www.ibtimes.co.uk/lloyds-group-partners-google-big-data-analytics-project-understand-customers-1547890)、花旗集团 [Sharethis](https://cloud.google.com/customers/sharethis/) 、 [Wepay](https://wecode.wepay.com/posts/bigquery-wepay) 、 [Zulily](https://engineering.zulily.com/2015/08/07/building-zulilys-big-data-platform/) 等等。

我们的许多客户每月不用支付任何费用，或者只需支付几便士或几美元——毕竟，只需 5 美元，你就可以获得多达 200，000 个查询。我们的许多客户都以 Pb 级规模运营。他们中的许多人甚至没有和我们说过话就达到了这个规模！而有的客户甚至达到 XXX PB 规模！

我们希望您加入这个不断增长的列表:)