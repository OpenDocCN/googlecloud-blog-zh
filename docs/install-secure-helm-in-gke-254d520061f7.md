# 在谷歌 Kubernetes 引擎(GKE)安装安全头盔

> 原文：<https://medium.com/google-cloud/install-secure-helm-in-gke-254d520061f7?source=collection_archive---------0----------------------->

在我的上一篇文章中，我谈了很多 Helm 的乐趣，以及为什么您应该花时间将它安装到您的 Kubernetes 集群中。如果你需要复习，现在就去阅读那篇文章吧。

[](/@jonbcampos/installing-helm-in-google-kubernetes-engine-7f07f43c536e) [## 在 Google Kubernetes 引擎中安装 Helm

### 当我第一次真正开始进入 Kubernetes 时，我会去寻找各种必要程序的 docker 图像…

medium.com](/@jonbcampos/installing-helm-in-google-kubernetes-engine-7f07f43c536e) 

在上一篇文章中，我们确实谈到了保护 Helm 和 Tiller 服务器的重要性，这个服务器负责将容器实际安装到您的 Kubernetes 中。你不会希望有人通过 Tiller 进入你的 Kubernetes 集群，并在你不知情的情况下开始安装各种容器。为了防止这种情况，我们需要进一步安装 Helm，并添加[传输层安全(TLS)](https://en.wikipedia.org/wiki/Transport_Layer_Security) 。

![](img/5490744583da2d8b3161f48709efee54.png)

照片由 [NeONBRAND](https://unsplash.com/@neonbrand?utm_source=medium&utm_medium=referral) 在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄

不过不要担心，这不会太痛苦，幸运的是，我已经完成了最难的部分。在这篇文章中，我们将看看如何为您的安全头盔安装创建必要的密钥，一些困扰互联网上许多人的陷阱，然后是如何安装头盔图表的快速示例，现在我们已经有了 TLS 安全设置。

*如果你还没有通读或者甚至没有阅读过本系列* *的第一部分，你可能会感到困惑，对代码在哪里或者之前做了什么有疑问。记住这里假设你正在使用*[*GCP*](https://cloud.google.com/)*和*[*GKE*](https://cloud.google.com/kubernetes-engine/)*。我将始终提供代码和如何测试代码是按预期工作。*

[](/google-cloud/kubernetes-day-one-30a80b5dcb29) [## 库伯内特:第一天

### 这是 Kubernetes 帖子的必选步骤之一。如果你对 Kubernetes 感兴趣，你可能已经读过 100 本了…

medium.com](/google-cloud/kubernetes-day-one-30a80b5dcb29) 

> 注意:Helm 网站确实包括许多关于安全 Helm 安装的[伟大文档](https://docs.helm.sh/using_helm/#securing-your-helm-installation)，但是我觉得很高兴看到在一个工作示例中所有东西是如何联系在一起的，以及如何帮助解决一些一直阻碍的问题。我建议[在某个时候通读他们的文档](https://docs.helm.sh/using_helm/#securing-your-helm-installation)。

# 创建 TLS 密钥

**警告** *警告* ***警告*** ！这是几乎所有人都搞砸的一步。所有这些都正确后，接下来的步骤就很简单了。

我们如何解决密钥创建混乱的问题？简单。我们真正理解关键的开发过程，然后它真的很简单。大多数人犯错误的地方是他们只是从互联网上复制半解释的脚本，然后不知道为什么事情会变糟。所以让我们开始了解关键的开发过程。

这一切都是从使用`openssl`命令构建我们的主认证机构(CA)密钥开始的。

```
openssl genrsa -out ca.key.pem 4096
```

这将生成一个证书颁发机构(CA)密钥，该密钥将用于签署该过程中的其他密钥。所以我们可以形象地想到这个，让我们把这个键标为`A`。

![](img/ad5658b7b87f6ed92f6d5c0428e624af.png)

密钥“A”—证书颁发机构密钥— ca.key.pem

准备好该密钥后，我们需要创建一个证书，用该密钥和有关贵公司/组织的各种信息进行签名。

```
openssl req -key ca.key.pem -new -x509 \
    -days 7300 -sha256 \
    -out ca.cert.pem \
    -extensions v3_ca \
    -subj "${SUBJECT}"
```

![](img/f9fc2f78748ff86a2c15acb29401e10f.png)

用密钥“A”和您的信息签名的证书— ca.cert.pem

接下来我们需要为 Tiller 创建一个键——管理舵图的舵服务器。您需要为每个 Tiller 主机创建一个关键点。在这种情况下，我们只有一个，所以我们只需要一个。

```
openssl genrsa -out tiller.key.pem 4096
```

![](img/4ff6044f82571bd3424133c6adb52d52.png)

密钥“B”—Tiller 主机密钥— tiller.key.pem

现在让人们困惑的部分。我们需要为 Tiller 主机密钥(密钥“B”)创建一个证书，该证书也由贵公司/组织签署，并由原始证书签署。这样做是因为将来您可能需要使密钥过期，但您仍然可以根据原始证书进行测试，原始证书以数学方式嵌入到您的签名证书中。

```
openssl req -new -sha256 \
    -key tiller.key.pem \
    -out tiller.csr.pem \
    -subj "${SUBJECT}"
openssl x509 -req -days 365 \
    -CA ca.cert.pem \
    -CAkey ca.key.pem \
    -CAcreateserial \
    -in tiller.csr.pem \
    -out tiller.cert.pem
```

![](img/8edafb6655ac2fdbbfdbabfce675a343.png)

用密钥“A”和密钥“B”签名的证书以及您的信息— tiller.cert.pem

最后，我们还需要为所有需要访问 Helm 的用户重复同样的过程。在这种情况下，我们只有一个用户，我们称之为“`Helm`”。如果出于安全需要，您有多个用户，只需重复这个过程:为用户创建一个密钥，然后将这个密钥与原始证书混合在一起，创建一个用户证书。

![](img/6bb390a9af199ec4ca1a398ce26e4591.png)

键“C”—舵主机键— helm.key.pem

```
openssl genrsa -out helm.key.pem 4096
```

以及`Helm`用户证书所需的代码。

```
openssl req -new -sha256 \
    -key helm.key.pem \
    -out helm.csr.pem \
    -subj "${SUBJECT}"
openssl x509 -req -days 365 \
    -CA ca.cert.pem \
    -CAkey ca.key.pem \
    -CAcreateserial \
    -in helm.csr.pem \
    -out helm.cert.pem
```

生成最终的`Helm`用户证书，该证书为用户签名，但数学上也包含第一个密钥。

![](img/0627fcae01141202e58e5581a90257a1.png)

用密钥“A”和密钥“C”签名的证书以及您的信息— helm.cert.pem

准备好所有这些文件，并正确地放在一起，接下来的步骤进行得很快。

# 安装带 TLS 的舵

现在一切都准备好了，我们只需要安装头盔。我们会给赫尔姆舵柄证书和钥匙以及原件`ca.cert.pem`。这很重要，因为将来各种各样的用户会带着他们自己的证书来到 Helm(在我们的例子中是`Helm`用户),想要访问我们的 Helm 实例。Helm 用数学方法对照`ca.cert.pem`检查用户的数学接受度，然后允许或拒绝用户。

```
helm init \
    --tiller-tls \
    --tiller-tls-cert tiller.cert.pem \
    --tiller-tls-key tiller.key.pem \
    --tiller-tls-verify \
    --tls-ca-cert ca.cert.pem \
    --service-account tiller
```

现在，我们安装的 Helm 实例已经知道了将来进行安全检查所需的所有信息。

# 用 TLS 装置检验你的舵

您可以通过让`Helm`用户尝试与 Helm 交互来验证 Helm 是否已安装并正确使用 TLS。我们只需要调用一个已安装的舵图表列表就可以做到这一点。如果成功，我们将看不到任何回应。如果不成功，则出现错误。

```
helm ls --tls \
    --tls-ca-cert ca.cert.pem \
    --tls-cert helm.cert.pem \
    --tls-key helm.key.pem
```

随着一个成功的响应，你已经看到这个 Helm 命令需要太多的参数，对于懒惰的编码者来说是不友好的。因此，我们将证书信息移动到我们的`helm home`目录中。

```
cp ca.cert.pem $(helm home)/ca.pem
cp helm.cert.pem $(helm home)/cert.pem
cp helm.key.pem $(helm home)/key.pem
```

现在，我们可以运行相同的命令和 Helm，并在将来的调用中假设证书信息。

```
helm ls **--tls # we just need to add --tls to our Helm commands!**
```

再次成功！现在让我们安装一个舵图。

# 安全安装舵图

如果你在我的上一篇 [install Helm(不安全地)文章](/google-cloud/installing-helm-in-google-kubernetes-engine-7f07f43c536e)中看到了如何安装 Redis，那么这个命令看起来会很熟悉。唯一的不同是我们现在必须给我们的舵命令添加`--tls`。其他都一样。

```
helm install stable/redis \
    **--tls \ # look! this is the only change!**
    --values values/values-production.yaml \
    --name redis-system
```

嘣！现在一切都应该正常了。显然，如果我包含一个工作示例，我可以让每个人都更容易做到这一点。接下来让我们看看。

# 使用提供的脚本快速安装安全头盔

现在我们已经完全了解了一切是如何运作的。让我们变得懒惰，通过把它放入一个我们可以运行并为我们处理所有步骤的脚本来[半]忘记一切。

```
$ git clone [https://github.com/jonbcampos/kubernetes-series.git](https://github.com/jonbcampos/kubernetes-series.git)
$ cd [~/kubernetes-series/helm/scripts](https://github.com/jonbcampos/kubernetes-series/tree/master/helm/scripts)
$ sh [startup.sh](https://github.com/jonbcampos/kubernetes-series/blob/master/helm/scripts/startup.sh)
$ sh [add_secure_helm.sh](https://github.com/jonbcampos/kubernetes-series/blob/master/helm/scripts/add_secure_helm.sh)
$ sh [add_secure_redis.sh](https://github.com/jonbcampos/kubernetes-series/blob/master/helm/scripts/add__secure_redis.sh)
```

如果您在 Google Shell 控制台中运行这些命令，那么您将会看到一切都已设置好，可以进行积极的开发了。刺激又轻松！

# 用证书观察这些陷阱

大多数人(包括我在内)似乎都会遇到以下问题。

*   混淆密钥(CA)和证书(CERT)以及证书签名请求(CSR)。如果您没有正确命名您的 CA、CERT 和 CSR，那么很容易混淆您正在签署哪个 CA、哪个 CSR 以及最终签署哪个 CERT。确保命名是有目的地设置的，这样你就不会把事情弄糟。
*   我发现你**有**设置签约主体，具体是常用名(CN)有这个作品。我最初尝试使用`-batch`命令，但是由于这个原因`helm init`命令一直失败。一旦我在主题中设置了 CN，一切都开始工作了。
*   当我们将证书移入`helm home`时，请注意头盔会有一些重命名。您需要确保移动了正确的证书和密钥。

# 更多关于安全头盔安装的最佳实践

如果你特别偏执，想要更进一步，这里有一些方法可以做到这一点。

*   您可以通过为 Helm/Tiller 专门创建一个名称空间来进一步细分 Tiller，从而提高安全性。
*   在我的例子中，我包括了一个专门针对 Helm/Tiller 的 Kubernetes 服务帐户。这有助于再次为你的头盔安装提供更多的安全性。
*   您可以继续使用基于辊的访问控制来保护您的舵/舵杆安装(RBAC)。我强烈推荐[去查阅文档，看看如何将 RBAC](https://docs.helm.sh/using_helm/#role-based-access-control) 添加到你的舵/舵杆装置中。

# 结论

这更多的是你对 Helm post 的“最佳实践”,这可能是你对我上一篇文章的期望。我认为这是专家推荐的路线。现在你应该已经在你的谷歌 Kubernetes 集群舵/舵安装和一个简单/安全的方式添加舵图表。使您的开发周期更短、更有效。

# 拆卸

在您离开之前，请确保清理您的项目，这样您就不会为您用来运行群集的虚拟机付费。返回到云 Shell 并运行 teardown 脚本来清理您的项目。这将删除您的集群和我们构建的容器。

```
$ cd [~/kubernetes-series/helm/scripts](https://github.com/jonbcampos/kubernetes-series/tree/master/helm/scripts)
$ sh [teardown.sh](https://github.com/jonbcampos/kubernetes-series/blob/master/helm/scripts/teardown.sh)
```

# 本系列的其他文章

[](/@jonbcampos/installing-helm-in-google-kubernetes-engine-7f07f43c536e) [## 在 Google Kubernetes 引擎中安装 Helm

### 当我第一次真正开始进入 Kubernetes 时，我会去寻找各种必要程序的 docker 图像…

medium.com](/@jonbcampos/installing-helm-in-google-kubernetes-engine-7f07f43c536e) [](/google-cloud/kubernetes-running-background-tasks-with-batch-jobs-56482fbc853) [## Kubernetes:使用批处理作业运行后台任务

### 当构建令人惊叹的应用程序时，有时您可能想要处理用户之外的动作…

medium.com](/google-cloud/kubernetes-running-background-tasks-with-batch-jobs-56482fbc853) [](/google-cloud/kubernetes-run-a-pod-per-node-with-daemon-sets-f77ce3f36bf1) [## Kubernetes:用守护进程集在每个节点上运行一个 Pod

### 我最初给这篇文章起的标题只是“守护进程集”,并假设它足以抓住要点…

medium.com](/google-cloud/kubernetes-run-a-pod-per-node-with-daemon-sets-f77ce3f36bf1) [](/google-cloud/kubernetes-cron-jobs-455fdc32e81a) [## 库伯内特:克朗·乔布斯

### 有时候你的工作不是事务性的。我们不再等待用户点击按钮让系统亮起来…

medium.com](/google-cloud/kubernetes-cron-jobs-455fdc32e81a) [](/google-cloud/kubernetes-dns-proxy-with-services-d7d9e800c329) [## Kubernetes:带服务的 DNS 代理

### 构建应用程序时，通常需要与外部服务进行交互来完成业务…

medium.com](/google-cloud/kubernetes-dns-proxy-with-services-d7d9e800c329) [](/google-cloud/kubernetes-routing-internal-services-through-fqdn-d98db92b79d3) [## Kubernetes:通过 FQDN 路由内部服务

### 我记得当我第一次进入 Kubernetes 时。一切都是崭新的、闪亮的、有规模的。当我继续的时候…

medium.com](/google-cloud/kubernetes-routing-internal-services-through-fqdn-d98db92b79d3) [](/google-cloud/kubernetes-liveness-checks-4e73c631661f) [## Kubernetes:活性检查

### 最近，我整理了一篇关于 Kubernetes 就绪性调查以及它对您的集群有多重要的文章…

medium.com](/google-cloud/kubernetes-liveness-checks-4e73c631661f) [](https://itnext.io/kubernetes-readiness-probe-83f8a06d33d3) [## Kubernetes:就绪探测

### 如果对这个特性有任何疑问，我写这篇文章是为了说明这不是一个…

itnext.io](https://itnext.io/kubernetes-readiness-probe-83f8a06d33d3) [](/google-cloud/kubernetes-horizontal-pod-scaling-190e95c258f5) [## Kubernetes:水平 Pod 缩放

### 通过 Pod 自动扩展，您的 Kubernetes 集群可以监控现有 Pod 的负载，并确定我们是否需要更多…

medium.com](/google-cloud/kubernetes-horizontal-pod-scaling-190e95c258f5) [](/google-cloud/kubernetes-cluster-autoscaler-f1948a0f686d) [## Kubernetes:集群自动缩放

### 自动缩放是 Kubernetes 的一个巨大的(并且已经上市的)特性。当你的网站/应用程序/应用程序接口/项目变得越来越大时，洪水…

medium.com](/google-cloud/kubernetes-cluster-autoscaler-f1948a0f686d) [](/google-cloud/kubernetes-day-one-30a80b5dcb29) [## Kubernetes:第一天

### 这是 Kubernetes 帖子的必选步骤之一。如果你对 Kubernetes 感兴趣，你可能已经读过 100 本了…

medium.com](/google-cloud/kubernetes-day-one-30a80b5dcb29) 

问问题？反馈？我很想听听你可能会遇到什么问题，或者这是否有助于你更好地理解。如果我错过了什么，也可以随意分享。我们都在一起！

[Jonathan Campos](http://jonbcampos.com/) 是一个狂热的开发者，喜欢学习新事物。我相信我们应该不断学习、成长和失败。我总是开发社区的支持者，并且总是愿意提供帮助。因此，如果你对这个故事有任何问题或意见，请在下面提出。在 [LinkedIn](https://www.linkedin.com/in/jonbcampos/) 或 [Twitter](https://twitter.com/jonbcampos) 上与我联系，并提及这个故事。