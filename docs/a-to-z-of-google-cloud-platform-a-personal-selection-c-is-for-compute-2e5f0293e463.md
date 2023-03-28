# 谷歌云平台的 a 到 Z 个人选择——C 代表计算

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-c-is-for-compute-2e5f0293e463?source=collection_archive---------0----------------------->

计算是任何云提供商提供的产品，他们倾向于提供一系列基于 CPU 或内存的实例类型。这是很重要的服务，但从表面上看并不特别与众不同。在表面之下，虽然有差异，你需要了解这些，以作出深思熟虑的选择。我喜欢谷歌云平台(GCP 的)计算是因为它在许多方面与众不同，以下是我脑海中经常出现的:

[定制机器类型](https://cloud.google.com/custom-machine-types/)

[可抢占的虚拟机](https://cloud.google.com/preemptible-vms/)

[透明维护](https://cloud.google.com/compute/docs/zones#maintenance)

[持续使用折扣](https://cloud.google.com/compute/pricing#sustained_use)

我不会谈论性能，虽然去年 GCP 发表了几篇博客文章[这里](http://googlecloudplatform.blogspot.co.uk/2015/07/Multi-million-operations-per-second-on-a-single-Google-Cloud-Platform-instance.html)和[这里](http://googlecloudplatform.blogspot.co.uk/2015/09/Google-Cloud-Platform-delivers-the-industrys-best-technical-and-differentiated-features.html)，他们为自己说话，我真的没有什么可补充的，这只是盲目的最有性能的 imho。另外，我试图保证不重复官方文件，而是给你我对事情的看法。如果你想深入 GCP 实例的性能方面，我建议从 [perfkit 基准测试](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker)开始

[定制机器类型](https://cloud.google.com/compute/docs/instances/creating-instance-with-custom-machine-type)是我多年来一直听到的一个问题，请这么做，不管我碰巧在用什么样的云，所以当谷歌交付时，我正在做拳头抽水的事情，我是一个英国人，所以后来很尴尬！除此之外，这真的很酷，因为许多从 DC 本地托管的人习惯于处理固定的披萨盒，实际上从来没有能够在 RAM & CPU 方面匹配他们的应用程序使用，根据我的经验，通常浪费 CPU 和 RAM。不要误解我的意思，在很多情况下，那些标准的实例类型是绝对好的，并且已经被选择来满足大多数用例。但是在那些你需要从你选择的实例类型中挤出绝对最大值的情况下，定制的机器类型就可以帮忙了。自定义机器类型允许您创建一个具有自定义数量的 vCPUs 和内存量的实例，您可以通过登录到控制台并使用滑块进行配置、使用 gcloud 从 CLI 传入配置或使用 API 来完成此操作。所有这些都很奇怪，但是如果你不知道，通过收集指标来理解你的应用程序是如何工作的，并理解它在 RAM & CPU 方面真正使用了什么是值得的。使用标准的实例类型可能更好，当您从应用程序实际使用的资源方面完全理解了您的应用程序之后，可能会转移到自定义类型。我看到了一个未来，拥有一长串愚蠢的实例类型不再是定制机器类型，而是成为一种规范，让它成为 GCP :-)

[可抢占的虚拟机](https://cloud.google.com/compute/docs/instances/preemptible)嗯，我喜欢这些，不，我真的喜欢！它们是最终寿命结束的虚拟机，最长可持续 24 小时，但在 24 小时后它们肯定会消失。这意味着您可以以比标准实例成本低 70%的低成本获得它们。不，你不需要为他们出价，看市场价格，成本是固定的，你知道它的寿命有限，不会超过 24 小时！这是一种获得廉价实例的简单方法，这些实例可用于需要补充计算的工作负载，并且可以应对它们将在短时间内终止的事实。如果您已经将应用程序设计为解耦的和/或能够在这些实例之一消失时不眨眼，那么这些类型的实例的用例是很棒的。您可以通过确保将持久性磁盘(PD)选项设置为不自动删除它们正在使用的磁盘来保留这些磁盘，以便另一个实例可以从它停止的地方继续。伟大的使用案例:

在批处理作业中运行工人任务，例如针对 [Dataproc](https://cloud.google.com/dataproc/preemptible-vms) / hadoop 集群

启动额外的实例来帮助处理发布/订阅队列中的消息

为 HPC 集群、渲染任务等提供额外的处理能力

[透明维护](https://cloud.google.com/compute/docs/zones#maintenance) —当我想到这一点时，我只会想到实时迁移有多棒。托管实例的服务器总是需要打补丁和维护，谷歌热衷于维护补丁级别和推出更新…

实时迁移会自动将正在运行的实例移动到同一区域中的另一台主机上。

对于那些依赖于启动并运行同一个实例的应用程序来说，重启意味着您实际上会遇到停机，实时迁移功能必须是一个关键的决策点。举个例子，想象一下你在一个专用的服务器上运行的游戏，关闭这个服务器基本上会让所有玩家停止使用这个服务器。您可以立即看到使用实时迁移的优势。

如果您愿意，还可以将您的实例设置为终止和重启！

[持续使用折扣](https://cloud.google.com/compute/pricing#sustained_use) —这是一种令人头疼的免费自动获得折扣的方式。它确实是它所说的。如果您让一个实例在一个月中运行超过 25%，那么您为该实例使用的每一分钟都会自动获得折扣。您的实例在该月运行的时间越长，应用的折扣就越大，对于整个月运行的实例，您可以获得高达 30%的净折扣。如果你正在运行一个实例，那么你不需要做出任何承诺，也不需要担心这个实例符合什么条件。