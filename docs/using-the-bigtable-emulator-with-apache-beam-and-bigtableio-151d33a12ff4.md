# 将 Bigtable 模拟器与 Apache Beam 和 BigtableIO 一起使用

> 原文：<https://medium.com/google-cloud/using-the-bigtable-emulator-with-apache-beam-and-bigtableio-151d33a12ff4?source=collection_archive---------1----------------------->

![](img/62aebc463acf94af832bf02d996daa0e.png)

在本地使用 Beam + Bigtable，没有云资源

[Cloud Bigtable](https://cloud.google.com/bigtable) 是一个非常棒的高性能分布式 NoSQL 数据库，可以存储数 Pb 的数据，但有时您可能需要测试它的少量数据。

仅仅为了这个而产生一个远程实例是很麻烦的，你将会为了一个小的测试而浪费一些金钱和云资源。

幸运的是， [Bigtable 附带了一个模拟器](https://cloud.google.com/bigtable/docs/emulator),可以非常方便地用少量数据进行简单测试，因为您可以在本地计算机上运行它。

模拟器在内存中工作，公开与 Bigtable 相同的接口，因此任何能够连接到真实 Bigtable 实例的系统也应该能够使用模拟器。

文档实际上非常清楚如何设置正确的环境变量来使用模拟器，只需首先启动模拟器，然后在您想要使用模拟器的 shell 会话中执行该操作:

```
$(gcloud beta emulators bigtable env-init)
```

这将添加一个名为`BIGTABLE_EMULATOR_HOST`的环境变量，其中包含连接到模拟器的详细信息。

模拟器是一个最初为空的内存数据库。您需要创建表，填充它们，等等。要创建表格，您可以[使用 cbt 实用程序](https://cloud.google.com/bigtable/docs/quickstart-cbt)。您需要用文件`~/.cbtrc`中的一些设置来配置它，包括项目和实例名。

但是您应该为模拟器使用什么项目和实例名称呢？

这很简单:任何。

无论您选择什么样的*项目*和*实例*值，如果您在 Bigtable 客户端中设置它，模拟器将访问该表，就像它们在那个*项目*和*实例*下一样。因此，要创建本地表，只需用如下值配置您的`~/.cbtrc`文件:

```
project = fake-project
instance = fake-instance
```

然后使用`cbt`实用程序创建表格。

如果您想使用 BigtableIO 从 Apache Beam 访问模拟器中的一个表，您必须使用与您的`~/.cbtrc`文件中相同的项目和实例名称。

如果您在定义了`BIGTABLE_EMULATOR_HOST`变量的 shell 中使用`DirectRunner`，BigtableIO 将尝试连接到模拟器。例如，下面的代码将尝试从模拟器中名为`mytable`的表中读取数据:

如果你想写而不是读，也是一样的道理。

如果您使用任何其他客户端，只需确保设置了环境变量`BIGTABLE_EMULATOR_HOST`，并使用您选择的项目和实例名称。所有的 Bigtable 库都检查该环境变量，如果它存在，它们将使用它连接到模拟器，而不是连接到远程实例。

结合 Bigtable 模拟器和 Apache Beam 的 DirectRunner，您可以在本地运行任何使用 Bigtable 的管道，而不必生成实例，也不必花费任何云资源来进行小型测试。