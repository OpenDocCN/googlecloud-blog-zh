# HTTPS 与 GKE 证书管理器

> 原文：<https://medium.com/google-cloud/https-with-cert-manager-on-gke-49a70985d99b?source=collection_archive---------0----------------------->

![](img/228eb42706b8d3004c58798faa852beb.png)

[米尔科维](https://unsplash.com/@milkovi?utm_source=medium&utm_medium=referral)在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上的照片

在某些时候，通常是在发布的时候，你开始担心像 HTTPS 这样的事情，以及如何向外界公开你的应用程序。在如何为您的域创建和管理您的证书方面，您有几个选项，但实际上最好的方式是您不必管理这些东西的方式—这意味着使用 Cert-Manager 并让在您的集群内加密。

我想指出的是，我在这篇文章中写的所有东西都在外面，我只是发现它们很模糊而且分散，所以在第一次终于弄清楚之后，然后一次又一次地重复这个过程一个多月，我觉得在一个地方写出来是有帮助的，并希望可以省去其他人在凌晨 4 点发布代码时的麻烦。

*如果你没有通读甚至没有读过本系列* *的第一部分* [*，你可能会感到困惑，对代码在哪里或者之前做了什么有疑问。记住这里假设你正在使用*](/@jonbcampos/kubernetes-day-one-30a80b5dcb29)[*GCP*](https://cloud.google.com/)*和*[*GKE*](https://cloud.google.com/kubernetes-engine/)*。我将始终提供代码和如何测试代码是按预期工作。*

[](/google-cloud/kubernetes-day-one-30a80b5dcb29) [## Kubernetes:第一天

### 这是 Kubernetes 帖子的必选步骤之一。如果你对 Kubernetes 感兴趣，你可能已经读过 100 本了…

medium.com](/google-cloud/kubernetes-day-one-30a80b5dcb29) 

# HTTP01 vs DNS01

如果您去阅读文档，[并且您应该在某个时间点](https://cert-manager.readthedocs.io)，您将看到有两种方法可以验证您是您的域的所有者，并且 Cert-Manager 应该将正确的证书安装到您的集群中: [HTTP01](https://cert-manager.readthedocs.io/en/latest/tutorials/acme/http-validation.html) 和 [DNS01](https://cert-manager.readthedocs.io/en/latest/tutorials/acme/dns-validation.html) 。

使用 HTTP01，您必须以正确的顺序设置所有内容，以便 Cert-Manager 和 Let's Encrypt 可以在颁发您的证书之前验证您的站点已启动并正在运行。此外，你**必须**在你的域名的根处有可用的**，否则**验证将失败**。**

使用 DNS01，您必须创建一个 GCP 服务帐户，并将信息提供给 Cert-Manager，以便让我们加密可以直接与您的 DNS 对话以验证该域。

我现在告诉你，DNS01 可能看起来更难，但它是更容易的方法。用 DNS01 就行了。

> 开始这个过程时，我假设您有一个带有部署和服务的 Kubernetes 集群，就是这样。显然，你可以有更多，但在本文的后面，我们将增加入口。好了，我们开始吧。

# 创建服务帐户

因为我们走的是简单的路线，使用 DNS01 验证，我们首先需要创建一个服务帐户。这是一个多步骤的过程，首先在 GCP 创建一个服务帐户，下载该帐户，并使用下载的帐户作为 Kubernetes 集群中的秘密。

首先，在 Google Cloud Shell 中创建一个服务帐户。

为您的 DNS 创建服务帐户

# 创建证书的秘密

下载完这个文件后，我们只需要把它变成 Kubernetes 集群中的一个秘密。

将服务帐户作为秘密安全地存储在 Kubernetes 集群中之后，您就可以进入下一步，为 Let's Encrypt 和 Cert-Manager 创建证书颁发者了。

# 安装舵和证书管理器

现在我们需要将 Cert-Manager 安装到您的 Kubernetes 集群中。为此，我们将[使用 Helm，因为它使复杂的安装变得轻而易举](/google-cloud/install-secure-helm-in-gke-254d520061f7)。如果你对 Helm 不熟悉，你应该看看关于这个问题的另一个帖子。

[](/google-cloud/install-secure-helm-in-gke-254d520061f7) [## 在谷歌 Kubernetes 引擎(GKE)安装安全头盔

### 在我的上一篇文章中，我谈了很多关于 Helm 的乐趣以及为什么你应该花时间把它安装到你的…

medium.com](/google-cloud/install-secure-helm-in-gke-254d520061f7) 

假设您的 Kubernetes 集群中安装了 Helm，我们可以用一行代码安装 Cert-Manager。神奇！

仅仅几行代码就能做这么多事情，这真的令人惊讶。我们已经完成了一半，最困难的部分都完成了！

# 申请发行人

使用“让我们加密”时，您需要提供“让我们加密”和证书颁发者的联系信息。奇妙的是，Cert-Manager 处理与 Let's Encrypt 的通信，所以只要您向 Kubernetes 集群提供一个 Issuer 文件，其余的就由您处理了。

让我们看看您需要创建的发行者文件。

这将创建一个准备和生产`ClusterIssuer`文件，供 Cert-Manager 使用。我们将只真正使用生产文件`ClusterIssuer`,但是如果您想使用它的话，另一个文件也不错。重要的是更新电子邮件地址，成为您组织的联系人。

有了这个文件，我们只需要将它应用到我们的集群。

申请发行人

很简单！下一步是创建实际的证书。

# 应用证书

现在是真正令人兴奋的部分——这也需要一点耐心。我们将创建一个新的证书，它将通过 Cert-Manager 发起一个请求，让我们加密以获得 TLS 证书。

与发行者一样，首先，我们创建 Kuberenetes 文件。

然后我们应用文件。

现在真正令人兴奋的事情开始发生了。您可以通过连续运行以下命令来观察进度。

我们一遍又一遍地调用这个函数，看着 Cert-Manager 代表我们与 Let's Encrypt 通信。如果你在`ClusterIssuer`或`Certificate`上搞砸了，这是你发现的时候了。通过阅读输出，我们可能会发现域设置不正确或其他一些问题。这些很容易修复和重新部署文件。每当您重新部署任何文件时，请确保也重新部署 certificate.yaml。

大约 10-15 分钟后，您将最终看到您所希望的最惊人的输出，一条成功的消息！

这真的很神奇！完成这些后，我们只需要几个简单的步骤，就可以放下凌晨 4 点的工作了！

# 创建一个全球 IP 地址

为了实现这一切，我们需要为您的 Kubernetes 集群创建一个固定的 IP 地址，以满足其所有的入口需求。GKE 提供给 Kubernetes 集群的默认 IP 地址是短暂的，这意味着它随时都会改变，所以这是行不通的。我们需要在 GCP 为您的集群创建一个静态 IP 地址。这真的很简单，只需要一行脚本。

这可能要花一点时间，但你会有你的 IP 地址路由到你的 DNS 更多关于这一点。

# 将您的域名指向您的 IP 地址

有了您的 IP 地址，我们现在需要转到您的 DNS，并将 DNS 名称设置为指向您的 IP 地址。出于我的需要，我使用云 DNS 作为 GCP 的一部分。我不打算重复这些步骤，因为[谷歌在记录快速入门](https://cloud.google.com/dns/docs/quickstart)的步骤方面做得很好。基本上，只需使用 CName 将您的 IP 地址指向 DNS 条目。

[](https://cloud.google.com/dns/docs/quickstart) [## 快速入门|云 DNS 文档| Google 云

### 无论您的企业是刚刚踏上数字化转型之旅，还是已经走上数字化转型之路，谷歌云的解决方案…

cloud.google.com](https://cloud.google.com/dns/docs/quickstart) 

# 在您的入口中包含 TLS

在整个过程中，我们假设您有一个部署和一个服务，现在我们添加入口资源以路由到您的服务。

这为您的主机建立了一条安全的路径，您的所有服务都可以利用由您的`my-certs`证书管理的秘密来使用这条路径。你完了！

# 结论

这是很多步骤，但它们是值得的。尤其是现在你的 SSL 证书将自动由 Cert-Manager 管理，并由 Let's Encypt 发布，你不必担心。您可以获得所有的安全性，而不会持续头痛。这是值得的一次性设置成本。

如果您阅读了这篇文章，并有任何问题，请随时联系我们。我知道这可能很复杂，我已经经历了这些步骤很多很多次了。

[乔纳森·坎波斯](http://jonbcampos.com/)是一个狂热的开发者，也是学习新事物的爱好者。我相信我们应该不断学习、成长和失败。我总是开发社区的支持者，并且总是愿意提供帮助。因此，如果您对这个故事有任何问题或评论，请在下面添加。通过 [LinkedIn](https://www.linkedin.com/in/jonbcampos/) 或 [Twitter](https://twitter.com/jonbcampos) 与我联系，并提及这个故事。