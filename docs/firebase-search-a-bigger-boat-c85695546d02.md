# 火力基地搜索:一艘更大的船

> 原文：<https://medium.com/google-cloud/firebase-search-a-bigger-boat-c85695546d02?source=collection_archive---------0----------------------->

Firebase 提供了用于管理数据流的查询功能，而不是用于分面搜索。但是你需要搜索你的用户、帖子、订阅等。跨多个属性。

问题:我究竟该如何搜索这些数据？？？

**回答:你需要一艘更大的船。**

# 弹性搜索和 Algolia FTW

Firebase 不提供搜索功能，因为其他服务已经完成了搜索。

认识一下 [Elasticsearch](https://www.elastic.co/) 和 [Algolia](https://www.algolia.com/) 。

这两个服务是非常非常大的船。

[**Elasticsearch**](https://www.elastic.co/) 是开源的，专门存储海量数据，并使其可以立即用 [Apache Lucene](https://lucene.apache.org/) 进行搜索。Elasticsearch 非常强大，你可以把它当作自己的 NoSQL 存储解决方案来使用。许多公司将他们所有的服务器日志直接提供给 Elasticsearch，以获得可搜索的存储。但我们大多数人用它来满足应用程序最奇特的搜索需求。

[**Algolia**](https://www.algolia.com/) 是一项针对网络和原生搜索的付费服务，强调易用性和 1 毫秒的搜索结果。你知道那些在你输入的时候可以看到一个建议列表的网络表单吗？那是阿尔戈利亚。这是我见过的最干净的开发服务之一。阿尔戈利亚队值得所有的道具。

# FirebaseSearch:数据同步

我已经写了一个小的 Node.js 包，并在 NPM 上发表为[颤动-火焰库-搜索](https://github.com/deltaepsilon/firebase-search)。FirebaseSearch 为 Elasticsearch 和/或 Algolia 提供 Firebase 参考和配置，并管理所有数据同步。

我刚刚花了两个小时写了一篇刻薄的自述。至少在开始实现之前扫描一下。

这里有一个基本的代码示例。注意，我有一个到端口 9200 的本地 ssh 隧道运行到我的 Elasticsearch 集群。我还隐藏了我的 Algolia 配置。我不会给你我的 api 密匙。

如果你想看整个过程，我也有一个 [YouTube 视频](https://youtu.be/JgbeoD3sTy8)可以看:

# 搜索您的索引

搜索索引最好留给个人实现，所以 FirebaseSearch 不提供任何类型的搜索功能。

Algolia 有一些很棒的嵌入式搜索栏。或者你也可以自己设计，从你的客户端浏览器/应用程序/任何东西直接连接到 Algolia。

Elasticsearch 提供一系列交钥匙服务。这些应该很容易启动和配置，但我没有使用过。我更喜欢使用 Google Cloud Launcher，并运行自己的服务器来通过安全隧道代理请求。我更喜欢它，因为它便宜。但这是额外的工作。

如果您正在运行自己的 Elasticsearch 集群，我建议不要搞乱集群的计算实例或配置。只需使用谷歌的云启动器启动它，然后在它前面安装一个服务器来提供访问。您可能需要对您的防火墙规则稍加修改，可能需要编写一些花哨的 Nginx 指令或 ssh 代理命令。实施起来并不野蛮。就是烦。我有一个非常复杂的 Nginx 配置的例子，我想出这个例子来只代理搜索请求通过我的集群。我应该使用 ssh 隧道。本来会更容易。

# 有问题吗？

请在推特上联系我，地址是[克里斯·埃斯普林](https://medium.com/u/aece6077659f?source=post_page-----c85695546d02--------------------------------)或 [YouTube](https://www.youtube.com/channel/UCaAByidxypZTYOj4OsKzD2Q) 。我很乐意回答问题并尽我所能提供帮助。