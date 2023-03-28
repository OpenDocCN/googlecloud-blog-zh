# 在本地测试容器时使用 Google Cloud 用户凭证

> 原文：<https://medium.com/google-cloud/use-google-cloud-user-credentials-when-testing-containers-locally-acb57cd4e4da?source=collection_archive---------0----------------------->

![](img/506799c1bd6a46ccf1431f55c5a5ece5.png)

容器打包如今非常流行:它允许执行环境的完全定制，并且是语言不可知的。越来越多的应用程序使用它。
现在，为了验证生产环境的行为，**开发人员需要测试容器**，而不仅仅是单元测试的工作量。

在某些情况下，**容器化的应用程序可能需要访问 Google Cloud API** ，从而获得认证。当**部署在 Google 云服务上时，** [**元数据服务器**](https://cloud.google.com/compute/docs/storing-retrieving-metadata) **可到达**，并提供[应用默认凭证(ADC)](https://cloud.google.com/docs/authentication/production) 。

> 如何通过 ADC 本地认证进行测试？

根据定义，**集装箱运行在一个隔离的环境中**。这意味着容器内部不知道您的本地配置，因此您的凭证不会被加载

# 服务帐户密钥文件解决方案

如果你用这个查询`google cloud container test authentication`继续谷歌搜索，**第一个链接会把你带到** [**云运行本地测试教程**](https://cloud.google.com/run/docs/testing/local) 。
这是一个很棒的教程，它解释了**如何在本地测试的容器运行时环境中加载凭证。**

***但是*** 建议使用的 **JSON 文件是一个服务账户密钥文件**。必须在本地生成和存储！

**这个解决方案不够安全**，正如[我以前的文章](/google-cloud/the-2-limits-of-iam-service-on-google-cloud-7db213277d9c)中所描述的，我想**避免服务帐户密钥文件**。此外，是 [Eran Chetzroni](https://medium.com/u/816233db7d13?source=post_page-----acb57cd4e4da--------------------------------) 的[大问题](/@chetz/thanks-guillaume-for-a-great-article-i-was-wondering-if-there-is-an-easy-way-to-develop-locally-85f2b2a93153)激发了这篇文章。

# 本地环境和用户凭据

当你在你的本地环境下编写你的应用程序时，你**使用符合你偏好的语言的谷歌授权库**。这个库可以**直接在你的代码**中使用，也可以**在服务特定的库**中作为依赖使用，比如云存储客户端库。

**Google auth library 试图通过按此顺序执行检查来获得有效的凭证**

*   **看环境变量**的`GOOGLE_APPLICATION_CREDENTIALS`值。如果存在，使用它，否则…
*   **看元数据服务器**(仅限 Google 云平台)。如果它返回正确的 HTTP 代码，就使用它，否则…
*   **查看“知名”位置**是否存在用户凭证 JSON 文件

“众所周知”的地点是

*   在 linux 上:`~/.config/gcloud/application_default_credentials.json`
*   在 Windows 上:`%appdata%/gcloud/application_default_credentials.json`

为了**在您的本地环境**上获得您的默认用户凭证，您必须使用`gcloud` SDK。您有两个命令来获得身份验证:

*   `gcloud auth login`获得所有后续`gcloud`命令的认证
*   `gcloud auth application-default login` **在本地“众所周知”的位置创建您的 ADC。**

两个命令都触发 OAuth2 认证流，同步或异步，并在本地存储刷新令牌。

# 将您的用户凭据加载到您的容器中

现在，我们有了拼图的两块

*   **JSON 凭证文件**。当然，不是服务帐户凭证，因此[2 个 IAM 限制适用于此处](/google-cloud/the-2-limits-of-iam-service-on-google-cloud-7db213277d9c)
*   运行容器时，**命令加载一个 JSON 凭证。*我就不多此一举了，* [*云运行教程解决方案*](https://cloud.google.com/run/docs/testing/local) *很棒！***

因此，您必须像这样运行您的本地`docker run`命令

```
ADC=~/.config/gcloud/application_default_credentials.json \
docker run \
<YOUR PARAMS> \
-e GOOGLE_APPLICATION_CREDENTIALS=/tmp/keys/FILE_NAME.json \
-v ${ADC}:/tmp/keys/FILE_NAME.json:ro \
<IMAGE_URL>
```

# 避免服务帐户密钥文件

同样，这里有一个简单的**解决方案来防止在本地环境中使用服务帐户密钥文件**。

然而，即使这个解决方案很棒，**也不要太想在本地环境之外使用它**。JSON 文件，即您的**用户凭证 JSON 文件或服务账户密钥文件，是“机密”**，需要作为机密处理。

因此，**永远不要将这些 JSON 文件添加到你的容器**中，尤其是如果它是公共的。容器只是一种打包模式，它不加密/隐藏任何东西！

**不要在本地和您自己的环境之外的其他环境中使用您的用户凭据**(如生产、试运行等)。API 访问将在您授权下代表您执行(并按原样登录)。
而且，因为它是用户凭证，所以您可能会被 2 IAM 限制阻止[。](/google-cloud/the-2-limits-of-iam-service-on-google-cloud-7db213277d9c)

所以，一如既往的，**玩安全的时候，明智的思考**！