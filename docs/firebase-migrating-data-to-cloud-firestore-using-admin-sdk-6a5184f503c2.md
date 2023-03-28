# Firebase:使用 Admin SDK 将数据迁移到云 Firestore

> 原文：<https://medium.com/google-cloud/firebase-migrating-data-to-cloud-firestore-using-admin-sdk-6a5184f503c2?source=collection_archive---------1----------------------->

![](img/b0b4fc751695f302787094ae7250a3eb.png)

去年秋天，谷歌推出了[云 Firestore](https://firebase.google.com/docs/firestore/)——一种高度可扩展的数据库服务，支持表达查询和实时监听。如果你是一名熟悉 [Firebase 实时数据库](https://firebase.google.com/docs/database/)的应用程序开发人员，如果你有兴趣尝试云 Firestore，你可能正在寻找一种方法将你现有的数据从实时数据库迁移到 Firestore。这篇文章阐明了这个主题，并展示了如何使用 [Firebase Admin SDK](https://firebase.google.com/docs/admin/setup) 为 Firestore 实现定制的数据迁移脚本。

我们需要提醒自己的第一件事是，实时数据库和 Firestore 基于两种截然不同的数据模型。实时数据库将数据存储在 JSON 树中。它很简单，非常适合存储任何可以编码成 JSON 的东西。相比之下，云 Firestore 是一个面向文档的数据库。它将数据存储在*文档*中，这些文档被分组到*集合*中。文档由键值对组成，它们可以存储几乎任何东西——包括原始字节。此外，每个文档都允许有子集合，这为 Firestore 数据模型增加了层次结构的概念。

由于数据模型的差异，没有一个通用的工具可以将数据从实时数据库迁移到 Firestore。尝试这种迁移的开发人员应该分析他们现有数据的模式，并将实时数据库中的 JSON 树映射到 Firestore 中的一组集合和文档。一旦确定了这种映射，就可以使用 Firebase Admin SDK 来实现实际的数据迁移。Admin SDK 支持这两种数据库服务。因此，我们可以使用它来实现 ETL 作业，从一个数据库提取数据，并加载到另一个数据库。

让我们考虑一个例子。假设实时数据库包含来自一个假想聊天应用程序的以下数据。

清单 1:实时数据库中的示例数据集

在 Firestore 中，我们可以将每个聊天室保存为名为`rooms`的集合中的一个文档。这些文档可以存储高级别的房间元数据以及会员信息。聊天消息可以存储在每个房间文档下创建的`messages`子集合中。这实际上产生了一个 Firestore 模式，它可以接受我们的聊天应用程序使用 Firebase 实时数据库进行的相同查询。

现在让我们实现我们的数据迁移脚本。我将使用 Firebase Python Admin SDK 来完成这项任务。Python Admin SDK 通过 REST(与 WebSockets 相反)与实时数据库进行交互。这使得它特别适合从实时数据库中批量下载数据。清单 2 显示了删除助手函数后的结果代码。包括所有实用程序的完整源代码可在[这里](https://gist.github.com/hiranya911/7cb8408bc24cfda39473b09244bc906c)获得。

清单 2:将数据迁移到 Firestore 的示例 Python 脚本

我们首先用服务帐户凭证初始化 Admin SDK。初始化后，我们可以通过编程来访问这两个数据库服务。Admin SDK 的`db`模块使我们能够从实时数据库中提取现有数据作为 Python 字典。然后，我们在内存中执行必要的转换，并将结果值加载到 Firestore。在写入时，我们将多达 500 个写入分组到同一个批处理操作中，以最大限度地减少 RPC 调用的数量(参见[完整实现](https://gist.github.com/hiranya911/7cb8408bc24cfda39473b09244bc906c))。为了更好地实现这一点，我们分两个阶段执行 Firestore 上传。首先我们写所有的聊天室，一次 500 个房间。然后我们写下所有的信息，一次 500 条。

清单 2 这样的定制脚本可以轻松地迁移中小型数据集。但是，它假设数据集可以容纳在内存中。在单次读取操作中，可以从实时数据库获取的数据量也有 256 MB 的限制。因此，在处理大型数据集时，可以考虑从 Firebase 控制台将数据导出到 JSON 文件中。然后从导出的文件执行一个脚本，并使用 Admin SDK 将数据增量上传到 Firestore。您可以采用相同的方法来迁移您的其他现有数据集(MySQL、CSV 文件、电子表格等。)到云 Firestore。

执行数据迁移时要考虑的另一个方面是成本。从实时数据库下载的[数据，以及上传](https://firebase.google.com/docs/database/usage/billing)到云 Firestore 的[数据，你都会收到账单。除了实际的长期存储成本之外，您可能还需要为网络带宽使用和读写的文档数量付费。对于小型数据集来说，这通常不是问题。但是，如果您是一个拥有大量数据的付费 Firebase 用户，在开始批量数据传输之前，请确保您理解这两种数据库服务的成本含义。](https://firebase.google.com/docs/firestore/pricing)

我希望这篇文章能提供一些关于将数据从 Firebase 实时数据库迁移到 Cloud Firestore 的实用技巧。官方 [Firestore 文档](https://firebase.google.com/docs/firestore/firestore-for-rtdb)也详细讨论了这个主题，你肯定也应该浏览一下。如果没有别的，我希望这给你一个窗口，使用 Firebase Admin SDK 从服务器端代码与实时数据库和云 Firestore 进行交互。祝您编码愉快，并随时分享您对这两种云数据库服务的体验。