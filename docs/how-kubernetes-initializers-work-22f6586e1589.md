# Kubernetes 初始化器如何工作

> 原文：<https://medium.com/google-cloud/how-kubernetes-initializers-work-22f6586e1589?source=collection_archive---------1----------------------->

如果我要指出 Kubernetes 起飞的一个原因，我可能会说是因为它的令人敬畏的社区。第二个原因是 [Kubernetes API](https://kubernetes.io/docs/reference/) 的灵活性，以及在其上编写定制扩展或插件是多么容易。在本文中，我将深入探讨一个新概念:**初始化器**，这是一种在实际创建 Kubernetes 资源之前对其进行修改的动态可插拔方式。

初始化器已经是 Kubernetes 1.7 中的 alpha 特性了。例如，我们在 [Google 容器引擎](https://cloud.google.com/container-engine/)使用初始化器来扩展 Kubernetes 特性库，你也可以通过实现新的初始化器来满足你的需求。

到目前为止，Kubernetes 只有[许可控制器插件](https://kubernetes.io/docs/admin/admission-controllers/)在创建资源之前拦截它们。例如，您可以让一个许可插件强制所有容器映像来自一个特定的注册表，并防止其他映像被部署到 pod 中。有相当多的[准入控制器提供诸如强制限制、应用预创建检查以及为缺失字段设置默认值等功能。](https://kubernetes.io/docs/admin/admission-controllers/)

准入控制器问题是:

1.  它们被编译成 Kubernetes:如果你要找的东西不见了，你需要分叉 Kubernetes，编写准入插件并自己保持一个分叉。
2.  您需要启用每个准入插件，方法是将其名称传递给 [kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver/) 的`--admission-control`标志。在许多情况下，这意味着重新部署集群。
3.  一些托管群集提供程序可能不允许您自定义 API 服务器标志，因此您可能无法启用源代码中可用的所有准入控制器。

[提出了动态/外部准入控制器](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/admission_control_extension.md)来解决这些问题。目前有两种类型的插件:初始化器和网络挂钩。初始化器类似于准入控制器插件，因为您可以在创建资源之前拦截它。它们不同于准入控制器插件，因为它们不是 Kubernetes 源代码树的一部分，也没有编译到源代码树中；你需要自己写一个控制器。

# 你能用初始化器做什么？

当您在创建 Kubernetes 对象之前拦截它们时，可能性是无限的:您可以以任何您喜欢的方式改变对象，或者阻止对象被创建。

以下是初始化器的一些想法，每个初始化器在集群中执行一个特定的策略:

*   如果 pod 有端口 80，或者有特定的注释，则将代理边车容器插入到 pod 中。
*   自动将带有测试证书的卷插入到测试命名空间中的所有 pod。
*   如果密码少于 20 个字符(可能是密码)，请防止创建它。

如果你不打算修改对象，只是为了读取对象而拦截，那么 webhooks 可能是获得对象通知的一个更快、更精简的替代方法。一定要看看这个基于 webhook 的准入控制器的例子。

我上面列出的一些功能，如注入边车容器或体积可以使用 [Pod 预设](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)来实现，灵活性较差。如果 Pod 预设确实适用于您，您可能不需要费心开发初始化器。

# 初始化剖析

1.  **配置哪些资源类型需要初始化:**[initializer configuration](https://kubernetes.io/docs/admin/extensible-admission-controllers/#configure-initializers-on-the-fly)对象让你配置哪些初始化器应该分配给哪些类型的资源。
    例如，你可以创建一个将“myproxy”初始化器添加到类型为`apps/v1beta1.Deployment`和`v1.DeamonSet`的对象中。您可以创建任意多的`InitializerConfigurations`,它们适用于所有的名称空间。
2.  **API 服务器将为新资源分配初始化器:**当您向 API server 提交部署对象时，它将更新部署的`metadata.initalizers.pending`并在那里添加“myproxy”值。这个字段显示当前分配给资源的初始化器。
    (准确的说，不是 apiserver 添加了初始化器。有一个名为“初始化器”的准入控制器插件，它使得整个初始化过程成为可能。它是通过向 kube-apiserver 添加`--admission-controller=Initializer`标志来实现的。)
3.  **您将编写一个控制器来监视资源:**您开发并部署到集群的这个定制控制器使用 Watch API 来监听新资源、捕获它们并进行所需的修改。
4.  **等待轮到你修改资源:**一旦你的控制器通过 Watch API 拦截了一个对象，它应该只修改在初始化列表的第一个元素上看到它的名字的对象(`metadata.initializers.pending[0]`)。否则，这意味着轮到其他初始化器修改资源了，它现在应该跳过修改。
5.  完成修改，让位于下一个初始化器。一旦您完成了对资源的修改，您的控制器应该从对象的`metadata.initializers.pending`列表中删除它的名称，并将对象保存回 API 服务器。
6.  **没有更多的初始化器，资源准备实现:**当 Kubernetes API 服务器看到对象没有更多的待定初始化器时，它认为对象“已初始化”。现在 Kubernetes 调度程序和其他控制器可以看到完全初始化的对象并使用它们。

您可以在集群上同时运行多个初始化器。这些定制控制器中的每一个都将得到关于他们订阅观看的资源(例如，pod)的修改的通知，但是他们将等待轮到他们来修改对象，直到他们在列表中看到他们的名字。

# 初始值设定项:在幕后

您可以开发和部署初始化器，而无需了解它们在 Kubernetes API server 中是如何实现的。让我更详细地讨论一下它是如何在 Kubernetes API 中实现的:

Kubernetes 社区在早期就已经开始考虑以钩子[的形式提供这样的扩展点。第一个](https://github.com/kubernetes/kubernetes/issues/3585)[在这里提出](https://github.com/kubernetes/kubernetes/pull/17305)，这个特性看起来有点类似于它实际上是如何实现的。[后面的提案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/admission_control_extension.md#initializers)阐明并解释了该功能的机制预期如何工作。如果你想了解它在幕后是如何运作的，一定要阅读链接的提议和[拉动请求](https://github.com/kubernetes/kubernetes/pull/36721)。

简而言之，当一个 Pod 资源被提交给 API 并被分配了一个未决初始化器列表时，它实际上将不会被调度，直到初始化完成。Kubernetes 调度器，调度器是另一个控制器，它监视 API 服务器中出现的 pod，并将它们分配给一个节点([了解更多关于调度的信息](https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work/))。

那么，为什么调度程序和其他控制器在这个对象初始化之前看不到它，即使这个对象保存在 API 服务器上(和 etcd 数据库中)并且对其他一些控制器(比如你的初始化器)是可见的？

答案是一个名为`includeUninitialized`的请求参数。该参数默认为 false，因此 API 在类似 WATCH 或 LIST 的请求中对默认客户端(例如`kubectl`)和控制器(例如调度程序)隐藏未初始化的对象。您开发的初始化器必须设置`?includeUninitialized=true`查询参数来观察这些对象。

初始化器阻塞创建请求。当创建对象的请求提交给 API 服务器时，它不会立即返回，请求会阻塞，直到初始化完成。如果您正在使用 kubectl，并且对象陷入未初始化状态，您会注意到`kubectl`在 30 秒后超时。

# 优点和缺点

[准入控制插件](https://kubernetes.io/docs/admin/admission-controllers/)编译成 Kubernetes API 服务器。要添加一个新的插件，您需要分叉 Kubernetes 源代码树，在其上开发您的插件并保持分叉。另一方面，[初始化器](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers)是在 Kubernetes 源代码树之外开发的。您可以使用 Kubernetes API 客户端轻松开发一个。它们像任何其他工作负载一样在集群上运行，您需要担心的事情更少。

**初始化器非常灵活:**一旦你在一个对象被实际创建之前就得到它，那就没有限制了。请注意，这种灵活性也可以让你很容易搬起石头砸自己的脚。你应该限制每个初始化器做一个任务，不要踩到对方的脚趾。

**编写一个初始化器很容易:**看看[这个例子](https://github.com/kelseyhightower/kubernetes-initializer-tutorial)由 Kelsey Hightower 编写，它基于一个注释向 Pod 添加了一个 sidecar 容器。它大约有 200 行 Go 代码，但是它很好地自动化了一项重要的任务。你现在就可以自己开发一个。但是如何开发一个产品级初始化器，并在产品中运行它呢？这可能会更有挑战性。

初始化器的正常运行时间很重要:当初始化器离线时，它仍然会被分配来初始化新的资源。这些资源将无限期地陷入“未初始化”状态，除非初始化器返回。这可能具有真实的现场含义:如果您有 pods 的初始化器，并且如果您的初始化器在扩展事件期间离线，则不会创建新的 pods，这可能会导致自动扩展操作失败并导致中断。

**现在还早:**在撰写本文时，初始化器目前在 Kubernetes v1.7 中处于 *alpha* ，而在 v1.8 中它的目标是 beta(更正:现在是 v1.9)。您需要在您的群集中启用 alpha 标志，以便从今天开始使用此功能。还要注意，我在本文中解释的许多东西可能不适用于该特性的稳定版本。

# 开发你自己的初始化器

最简单的入门方法是派生出 Kelsey Hightower 的初始化器示例,它是用 Go 编写的，根据注释的存在向部署对象添加一个 sidecar 容器。

在[源代码](https://github.com/kelseyhightower/kubernetes-initializer-tutorial/blob/2645c93e05f281c251ae7d835c7bb93dfdded0b8/envoy-initializer/main.go)中，您应该注意一些事情:指定`IncludeUninitialized=true`并以所有名称空间为目标的列表/观察函数、通知器及其重新同步周期、API 对象如何被克隆/变异，以及如何使用补丁执行更新

开发初始化器时要注意的事情:

*   确保你的初始化式不会出错。我之前提到过，你能做的最好的事情就是确保你有一个活性探测器，并对其进行监控/报警。如果你有所有 pod 的初始化器，这是特别需要的。
*   先有鸡还是先有蛋的问题:如果你有 pods/deployment 的初始化器，部署初始化器可能会阻塞，因为它不能初始化自己。您需要手动指定待定初始值设定项列表，以清空 pod 清单中的数组值(`[ ]`)。
*   初始化器是正常的工作负载，但是您应该将它部署到与正常工作负载不同的命名空间。内置的`kube-system`名称空间可能是存放初始化器的好地方。
*   初始化器可能会收到不完整的对象:例如，一个尚未被调度的 Pod 可能会丢失一些字段，比如“节点名”或“状态”。你应该用这个假设来测试初始化器。
*   初始值设定项可能以不同的顺序应用:待定初始值设定项的列表可能每次都有不同的顺序，您的实现应该可以很好地处理这个问题。
*   确保你的初始化器快速处理对象。初始化会阻止创建资源的请求。

# 立即获取启用初始值设定项的集群

在 Initializers 特性变得稳定之前(这使得它们在默认情况下被启用)，您可以通过运行以下命令从 Google 容器引擎获得 alpha 集群:

```
gcloud container clusters create my-cluster \
 --enable-kubernetes-alpha \
 --cluster-version 1.7.4
```

您可以使用这个集群来部署您的第一个初始化器。请注意，阿尔法集群将在 30 天后删除自己。

# 进一步阅读

如果初始化器对您来说是一个有趣的话题，请查看下面链接的参考资料。初始化器可能是扩展 Kubernetes API 最实用和最简单的方法。他们非常灵活，我很想听听你会提出什么样的想法。

**感谢** [**阿帕娜辛哈**](https://twitter.com/apbhatnagar) **和** [**许超**](https://github.com/caesarxuchao) **审阅本文草稿。**

*最初发表于*[*Ahmet . im*](https://ahmet.im/blog/initializers/)*。如果你喜欢这篇文章，你可以在 Twitter 上关注我，通过电子邮件订阅我的博客(不超过一篇文章/月)。*