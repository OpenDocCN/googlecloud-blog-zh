# 为什么是 Tensorflow 和 Kubernetes？—秘制酱

> 原文：<https://medium.com/google-cloud/why-tensorflow-and-kubernetes-secret-sauce-2a0dd4c0b157?source=collection_archive---------1----------------------->

因为当你倾听时，你会进步。

对于那些一直生活在图书架或服务器架下的人，让我们先做一个简短的介绍。

[Tensorflow](https://www.tensorflow.org/) :一个帮助你理解数据的库

[Kubernetes](https://kubernetes.io/) :帮助管理计算资源的平台

因此..他们有什么特别之处？我们来分解一下。

# 口径

两者都是一个系统的一部分，这个系统已经证明了它在这个领域的能力。Borg 为 Google 的基础设施提供动力，它是 kubernetes 的灵感来源，在很多方面也是其父。如果 Kubernetes 没有类似的功能，您可以期待更好的功能。另一方面，Tensorflow 实际上被谷歌用来改善现有的机器学习能力，并建立一些非常棒的新功能。

# 开放源码

是啊！他们两个都是。这看起来似乎是一件小事，但是开源项目的一个重要方面是他们依靠反馈而发展。如果你没有在你的项目中打开问题，你已经死了。这两个项目不仅得到了各自社区的好评，还拥抱了开源世界。持续的迭代，完整的反馈循环，质疑现有方法和特性的能力，以及推动社区的变化，这些都是这两个项目成为各自领域品牌的原因。

# 哲学目标

两个项目都信奉一种哲学。对 Kubernetes 来说，这是一个原则:任何可能失败的事情都会失败。所以开始让事情恢复正常。你从一个失败的状态开始，然后系统尽最大努力让你进入期望的状态。其他一切只是你如何实现它。另一方面，对于张量流来说，现有的机器是远远不够的。所以，为什么不利用它来[建立一个你期望的流程](https://www.tensorflow.org/how_tos/graph_viz/)，然后让 Tensorflow 来管理执行。这使得它变得分布式，这使得每个人的生活更容易

# 支持

不仅是个人和谷歌，这两个项目都获得了社区和其他行业领袖的大力支持。Kubernetes 现在是 Linux Foundation 下的[Cloud Native Computing Foundation](https://www.cncf.io/)的一部分，旨在简化和提供更好的云工具。Tensorflow 在谷歌内部引起了如此大的兴趣，以至于他们正在建立 [Tensorflow 处理单元](https://cloudplatform.googleblog.com/2016/05/Google-supercharges-machine-learning-tasks-with-custom-chip.html) (TPUs)，这些处理单元基本上是为运行 Tensorflow 而优化的硬件。不仅仅是谷歌，还有[英特尔和其他公司](https://blog.google/topics/google-cloud/google-and-intel-announce-strategic-alliance-accelerate-cloud-adoption-enterprise/)也致力于帮助 Tensorflow 取得成功。

# 服务/合作伙伴

既然一个项目是开源的，并不意味着你不能围绕它构建很好的服务。谷歌提供 Kubernetes 服务已经有一段时间了，名为[谷歌容器引擎(GKE)](https://cloud.google.com/container-engine/) 。还有其他产品。不仅仅是为了 kubernetes，也是为了与之一起工作的服务。至于 Tensorflow，谷歌云平台有一个[机器学习平台。同样，基于 Tensorflow 的机器学习模型的数量也在快速增长。预计它还将推出一个更简单、更高级别的 API 来学习和构建模型。](https://cloud.google.com/ml/)

我希望这篇文章是有见地的。如果你还有疑问，为什么不[直接问乡亲们](https://twitter.com/googlecloud/status/830184763260538884)？

你以为这一切都结束了，是吗？好吧，我有临别礼物给你。

*   向库本内特先生本人学习库本内特
*   [不用博士学 Tensorflow 和深度学习](https://codelabs.developers.google.com/codelabs/cloud-tensorflow-mnist)

尽情享受吧！别忘了告诉别人这件事。我们一起成长。:)