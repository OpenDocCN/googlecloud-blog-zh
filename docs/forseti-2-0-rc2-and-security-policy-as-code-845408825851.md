# 凡赛堤 2.0 RC2 和安全策略代码

> 原文：<https://medium.com/google-cloud/forseti-2-0-rc2-and-security-policy-as-code-845408825851?source=collection_archive---------2----------------------->

![](img/ab16805eac786f1de18dea9622b18c27.png)

不久前，我写了一篇关于凡赛堤安全快速安装的文章。我有机会尝试 2.0 版本，并想在这里记下一些事情，谈谈我是如何将安全策略作为代码来尝试的。

[的安装](https://docs.google.com/document/d/1RDw8QLhJVd-EAwIAviaboXwLoHPehteRGLCOXzEHBoc/preview)是相当无缝的，我不会进入太多的细节。在幕后，安装脚本做了一堆[部署管理器](https://cloud.google.com/deployment-manager)的事情。同样，在较高层次上，它创建了一些服务帐户、Google Cloud SQL 实例、Google Cloud SQL 数据库、存储桶和 Google Compute Engine 虚拟机实例。

凡赛堤 2.0 RC2 体系结构发生了变化，创建了一个客户端和一个服务器计算实例。这本身确实是一个安全考虑。您可能有这样一个用例，您希望允许某人访问运行凡赛堤扫描，而不允许访问服务器本身。现在，您可以在客户端做到这一点。

# 代码形式的安全策略(&配置)

在目前的状态下，我只是在管理规则。我会进行配置，但我有一些秘密要处理。也就是说，配置将以几乎相同的方式完成。

设置是 ***简单的*** ，我有一个 GitHub [项目](https://travis-ci.org/lzysh/ops-forseti-security)的配置和规则连接到[特拉维斯配置项](https://travis-ci.org)，另一个配置项将工作，但我更喜欢这种类型的工作特拉维斯配置项。您用于 Travis CI 的服务帐户将需要以下角色:

计算机管理员*(也许可以精简一些..)*
服务账户用户

这将允许服务帐户 ssh 到服务器并执行:

```
/home/ubuntu/forseti-security/setup/gcp/scripts/run_forseti.sh
```

这个脚本将从 bucket 中复制规则，bucket 是从 github 项目中复制的 [push.sh](https://github.com/lzysh/ops-forseti-security/blob/master/push.sh) 的前一部分。服务帐户还需要适当的存储桶权限。

我可以看到一些排序或规则/配置验证引擎对于这种类型的方法是有价值的，例如，通过指向新配置规则的客户端运行的方法。在配置到达服务器之前对其进行测试和验证。现在我只是用 yamllint 来验证 yaml，这样我至少可以确定我没有中断我的 yaml。

这就是我们的想法，你可以看到一个拥有广泛政策的社区是多么有价值。虽然有些策略可以满足您组织的特定需求，但其他策略可以是通用的/可重用的和共享的。抓住你需要的政策，将它们提交到你自己的回购中，推出它们，执行一次 forseti 运行。容易的事..