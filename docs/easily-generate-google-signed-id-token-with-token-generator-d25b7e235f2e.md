# 使用令牌生成器轻松生成 Google 签名的 id 令牌

> 原文：<https://medium.com/google-cloud/easily-generate-google-signed-id-token-with-token-generator-d25b7e235f2e?source=collection_archive---------1----------------------->

![](img/86cd772e827a2887b349c6a10c86cfba.png)

安全性对云提供商来说是一个巨大的挑战，谷歌云平台选择了**用 OAuth2 协议**来保护其所有服务。安全令牌在 API 调用的头中传递，用于标识用户。授权基于角色，并在 IAM 中进行检查。

这个令牌可以**标识一个用户账户，也可以标识一个服务账户**。在机器对机器的调用中，使用的就是这种类型的身份验证。

# 使用 gcloud 命令生成 Id 令牌

当您尝试访问服务时，如私有云运行或私有云功能，需要一个**签名的 id 令牌**。使用您的**用户帐户**，可以使用 gcloud SDK 轻松生成该令牌

```
# Authenticate with your service account
gcloud auth login # Generate an id token
gcloud auth print-identity-token
```

或者用**服务账号密钥文件**

```
# Load the service account identity
gcloud auth activate-service-account --key-file=key.json# Generate an id token
gcloud auth print-identity-token
```

一个特殊的技巧，**你可以模拟一个服务帐户**并像这样使用它

```
# Generate an id token
gcloud auth print-identity-token \
--impersonate-service-account=SA@PROJECT_ID.iam.gserviceaccount.com
```

*将* `*SA*` *替换为服务帐户名，将* `*PROJECT_ID*` *替换为您的项目 ID*

# 在 Google Cloud 环境下生成一个 Id 令牌

在 GCP 上，**每个组件都有自己的身份**，这意味着一个服务帐户会自动附加到组件上。**该服务帐户可定制，以实现更好的权利分离**，并在每个组件上应用**最小特权原则**。

如果没有指定服务帐户，则使用默认服务帐户，通常使用以下模式计算默认服务帐户:
`<PROJECT_NUMBER>-compute@developer.gserviceaccount.com`

因此，**您不需要为使用特定的服务帐户生成服务帐户密钥** **文件**。您可以通过[元数据服务器](https://cloud.google.com/compute/docs/storing-retrieving-metadata)直接使用**组件标识**。这里以[云运行](https://cloud.google.com/run/docs/authenticating/service-to-service#go)为例

***注意*** *:当你在 GCP 环境下，我强烈建议* ***千万不要*** *使用服务账号密钥文件。他们是*

*   *无用感谢组件标识*
*   *难以保全(或者存放在* [*秘密总管*](https://cloud.google.com/secret-manager/docs) *)*
*   *难以追踪*
*   *难以定期轮换(建议每 90 天一次)*

# 从 GCP 境外生成 Id 令牌

然而，当你不在 GCP 上，你想用一个自动平台生成一个 id_token(我的意思是没有用户账号)，**你必须有一个服务账号密钥文件**。

这在几种情况下发生:

*   您必须在 CI/CD 平台中与 GCP 服务进行交互
*   您有一个托管在 GCP 之外的应用程序(本地或其他云提供商)
*   开发人员没有一个谷歌帐户，你可以在 IAM 授权

当**开发者不熟悉 GCP 或者如果他们使用旧的框架**时，很难生成 id 令牌。我最近处理了两个用例:

*   开发人员希望使用 Postman 来测试云运行私有 API。不安装 gcloud。他们很难为他们的测试建立一个简单的过程
*   。Net 4.0 框架，并且缺少一些构建 JWT 令牌的方法，它们必须手动实现。

*最后一种情况可能出现在任何旧的语言和/或框架中。* ***在应用程序现代化方面，这可能是一个挑战和阻碍*** *。*

# 用于生成 id 令牌的令牌生成器

为了帮助开发人员并防止任何项目延迟，我实现了一个小工具， [token-generator](https://github.com/guillaumeblaquiere/token-generator) ，它基于参数中提供的服务帐户密钥文件生成 id_token。

这个工具公开了一个可以在本地主机上访问的网络服务器。想法是**简单地在这个服务器上执行一个** `**GET**` **查询来获得一个有效的令牌 ID** 。

*   开发人员在他们的本地环境中启动 token-generator，简单地在 localhost 上执行 get，并将授权头中的结果用作承载令牌。
*   旧的应用程序/框架在服务器的一个空闲端口上启动 token-generator 作为辅助程序，并在 localhost 中执行一个简单的 GET 来获取令牌并在后续查询中使用它

# 在任何地方使用令牌生成器…

**我** [**在 GitHub**](https://github.com/guillaumeblaquiere/token-generator) 上开源了这个小而有用的工具。它是用 Go 编写，可以在几个平台上编译(现在每个版本都自动生成 3 个)。

因为有了 Go，**流程轻便高效，对性能没有影响**。当您有一个生成 id 令牌的阻塞点时，您可以随意使用。

# …而且要小心

但是这个工具**并不能防止滥用服务账户密钥文件**，尤其是像 Git 这样的配置管理工具中的存储。

另外，**你绝对不可以在互联网上公开暴露这项服务。**为本地运行而设计，与其他软件搭配使用。

# 欢迎反馈

我将很高兴**得到反馈，并讨论您的用例**，引导您使用或不使用该工具。你也可以在 GitHub 上[打开问题。](https://github.com/guillaumeblaquiere/token-generator)