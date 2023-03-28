# 使用云扳手和 Java

> 原文：<https://medium.com/google-cloud/working-with-cloud-spanner-and-java-16e44ebc63b6?source=collection_archive---------3----------------------->

我们已经在以前的帖子中讨论了 Google Cloud Spanner 的架构细节，现在是时候更深入地讨论使用 Google Cloud Spanner 构建应用程序的细节了。

如果您决定在 Cloud Spanner 上构建您的应用程序，您可以依赖 ANSI 2011 SQL 支持和多种语言的客户端库。有很棒的教程可以帮助你入门，尽管它们没有深入讨论使用 Java 时的不同选项；通过客户端库的数据操作语言或变体，或者通过两个 JDBC 驱动程序的 SQL/DML。

我不打算深入讨论这些概念，但是我希望提供足够的信息来帮助您理解作为使用 Cloud Spanner 的 Java 开发人员的不同选择。为了确保我得到了正确的细节，这篇文章和为它编写的代码[(你可以在这里克隆)](https://github.com/helix-collective/spanner-dml-mutations-examples/tree/master)得到了 Java 专家[彼得·龙格](/@prunge_24023)(更恰当的说法是 [prunge-helix on Github](https://github.com/prunge-helix) )的技术支持

我们将使用与 Google Cloud Spanner 入门指南相同的模式，这在 Cloud Spanner 文档中的[模式和数据模型](https://cloud.google.com/spanner/docs/schema-and-data-model#creating-interleaved-tables)页面上有详细解释。

我们实际上是在创建一个音乐应用程序，我们的目录包含了歌手及其专辑的详细信息。歌手和专辑之间强大的父子关系非常适合一种独特的云扳手优化，称为交叉表，这在该页面上有描述，非常值得理解。

# 示例和选项

# ORMs 和 JDBC 司机

如果你是一个经验丰富的 Java 程序员，使用 ORM 或 [JDBC 驱动](https://cloud.google.com/spanner/docs/jdbc-drivers)与 Cloud Spanner 交互可能更容易或更相关。

ORMs 还可以让你用自己选择的语言在 Cloud Spanner 中操作数据变得更加容易，而不必编写 DML。在许多情况下，这些都是现有 Cloud Spanner APIs 的包装。例如在带有 spring 的 Java 中，spring-Cloud-GCP-starter-data-Spanner 使用云 Spanner API(com . Google . Cloud . Spanner . *)来执行语句。

当遵循现代编程实践时，与在代码中散布 DML 相比，使用 ORM 与数据库进行交互要容易和一致得多。因为 ORM 经常使用现有的客户机库，所以使用 DML 的所有好处都要比使用变体的好处多。得到了维护。

对于带有 SpringData 的 ORM，我们将首先创建歌手表:

现在我们将创建相册表:

当然，我们必须创建接口:

现在我们可以使用这些表格:

有两个 JDBC 驱动程序，包括一个由谷歌编写的开源驱动程序。它利用客户端库连接到 Cloud Spanner，并允许您执行 SQL 和扩展 DML。

如果您的语句需要在执行前将许多对象保存在内存中，那么使用 JDBC 驱动程序对数据库执行语句可能会更有效。需要多个连接、分组和聚合的大型语句可能很难用面向对象的方式管理，而编写包含这些操作的单个 DML 语句可能更简单。但是，从执行的角度来看，后一个例子在 ORM 和 DML 中不会大致相同

当然，如果您正在连接一个现成的应用程序，最简单的集成可能是通过 JDBC 驱动程序进行连接。

# 关于 SQL/DML 的快速说明

Cloud Spanner 支持 ANSI 2011 兼容 SQL，使您能够使用声明性 SQL 语句查询数据库，这些语句指定了您想要检索的数据。

有 [SQL 最佳实践](https://cloud.google.com/spanner/docs/sql-best-practices)可以帮助 Cloud Spanner 以最高效的方式找到相关数据，了解 Cloud Spanner 如何执行 SQL 语句可以大大提高性能。例如，使用参数和辅助索引是提高查询性能的两种方法。

# 数据操作语言(DML)和分区 DML

DML 可用于在云控制台、gcloud 命令行工具和客户端库中插入、更新和删除语句。DML 是为事务处理设计的，其中[分区 DML](https://cloud.google.com/spanner/docs/dml-partitioned) 是为批量更新和删除设计的，对并发事务处理的影响最小。在分区 DML 中，这是通过对键空间进行分区并在单独的、较小范围的事务中的分区上运行语句来实现的。

DML 语句在读写事务内部执行，只在您访问的列上获取锁。对于读取，共享锁用于确保一致性，写入或修改会导致排他锁。

以下 [DML 最佳实践](https://cloud.google.com/spanner/docs/dml-best-practices)将有助于提高性能，并最大限度地减少锁定。

现在，我们将通过使用 Java JDBC 来执行 DDL 和 DML 语句，来执行 ORM 示例中说明的相同步骤

突变

一个突变代表一系列的插入、更新和删除，Cloud Spanner 原子地应用到 Cloud Spanner 数据库中的不同行和表。这些是通过[突变 API](https://cloud.google.com/spanner/docs/modify-mutation-api) 执行的。

虽然您可以通过使用 [gRPC](https://cloud.google.com/spanner/docs/reference/rpc/google.spanner.v1#google.spanner.v1.CommitRequest) 或 [REST](https://cloud.google.com/spanner/docs/reference/rest/v1/projects.instances.databases.sessions/commit) 来提交突变，但是更常见的是通过客户端库访问 API。

如果你想更深入地研究这个话题，Peter Runge 将在下周发表一篇关于 DML 和突变的文章。

由于这是第三个例子，我们将假设您已经创建了表，并通过使用突变 API 向我们的歌手和专辑表添加数据来节省一些时间

如果你只是想使用标准客户端库，那么[入门指南](https://cloud.google.com/spanner/docs/getting-started/java)会带你浏览我们下面引用的同一个例子，代码是发布在 github 上的

ORM 和 JDBC 驱动程序也使用客户端库，所以您也可以使用它们来执行 DDL: