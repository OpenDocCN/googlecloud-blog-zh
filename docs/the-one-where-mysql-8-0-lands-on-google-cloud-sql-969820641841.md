# MySQL 8.0 登陆谷歌云 SQL 的那个

> 原文：<https://medium.com/google-cloud/the-one-where-mysql-8-0-lands-on-google-cloud-sql-969820641841?source=collection_archive---------0----------------------->

![](img/4488fa20c0d1609212d1067778f8a959.png)

[阿达·多格雷丝](https://www.instagram.com/ada_dpglace)和[莉莉·格雷斯](https://www.instagram.com/lilygramsrocks)。由[安东尼·费拉拉](https://www.twitter.com/ircmaxell)拍摄。

有很多事情让我开心。小狗(见图)、食物、酒和数据库……(排名不分先后)。还有让我更开心的事情，比如设计良好的模式和 ORM(对象关系映射)的正确使用。

MySQL 是我曾经又爱又恨的数据库。我越来越喜欢它，事实上，为了使它更加一致和更加现代化，我已经很久没有在日常项目中使用其他开源数据库了。怀着巨大的期望和钦佩，我为云 SQL 团队实现这一里程碑而自豪:

**今天，我们将在云 SQL** 上发布 *MySQL 8.0。它不仅是我们托管数据库产品组合中的一个新版本，而且还附带了所有云 SQL 功能，例如[自动存储增加](https://cloud.google.com/blog/products/gcp/digging-in-on-cloud-sql-automatic-storage-increases)、[高可用性](https://cloud.google.com/sql/docs/mysql/high-availability)、[跨区域复制](https://cloud.google.com/sql/docs/mysql/replication/cross-region-replicas)和 [PITR](https://cloud.google.com/sql/docs/mysql/backup-recovery/pitr) (时间点恢复)。更好的是:这不是测试版，而是正式发布！见[公告](https://cloud.google.com/blog/products/databases/mysql-8-is-now-on-cloud-sql)。*

你可以从零开始使用 today，也可以将现有的 MySQL 数据库迁移到云 SQL 作为一种最大限度减少停机时间的方法，你可以从谷歌云控制台或通过 gcloud 命令行工具使用外部复制 T21 功能。

MySQL 8.0 有一个巨大的新特性列表，你可以做一系列新的操作和查询。你甚至可以用 SQL 做一些愚蠢的事情，比如斐波那契数列([这是真的！](https://gist.github.com/gabidavila/4cbdbe1a5a0c038dd2a28e85b37c9b51#file-fibonacci_cte-sql))、[遍历一棵二叉树](https://gist.github.com/gabidavila/67111ce4cb032ae7ea0f92d1a6f1bd6f)甚至做旧的[嘶嘶作响](https://en.wikipedia.org/wiki/Fizz_buzz):

要点上的嘶嘶声

更严肃地说，值得企业关注的是，您现在可以避免子查询([或其他 N+1 问题](https://gabi.dev/2018/02/06/ramblings-on-optimizations-anti-patterns-and-n1/))，拥有更好的访问控制，并使用窗口函数。有很多新东西，这是我自 2017 年预览版以来关于 MySQL 8.0 的一小部分演讲。下面是我最喜欢的一个例子:

MySQL 8.0 新特性！

在谷歌，我们将任何使用键盘从事产品技术方面工作的人定义为*技术从业者。*这是一个更广泛的定义，但对数据库管理员、开发人员和系统管理员等来说更具包容性。

我的特殊目标一直是帮助日常从业者做他们想象不到他们的数据库能够做的事情。这就是我有[办公时间的原因。](https://gabi.tips/office-hours)这就是我为什么做这些演讲，并且应该写更多博客的原因。

如果你想了解更多关于 MySQL 8.0 的信息，我建议你阅读我的网站: [gabi.dev](https://gabi.dev) ，在那里[我发布了一些关于 MySQL](https://gabi.dev/?s=mysql+8) 的东西。

发布日快乐！