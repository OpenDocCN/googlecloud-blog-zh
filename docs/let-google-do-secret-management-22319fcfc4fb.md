# 让谷歌做秘密管理

> 原文：<https://medium.com/google-cloud/let-google-do-secret-management-22319fcfc4fb?source=collection_archive---------0----------------------->

在这篇文章中，我将谈论谷歌的 [Secret Manager](https://cloud.google.com/secret-manager/) 服务，并展示在 GCP 管理密码/令牌是多么容易。

![](img/efa5e9bc36518f6b6cdc22e6efad47ad.png)

照片由[斯蒂芬·斯坦鲍尔](https://unsplash.com/@usinglight)在 [Unsplash](https://unsplash.com/) 上拍摄

# 问题

管理机密和共享机密(如数据库密码、令牌等..)是一个难题，无法保护令牌会导致一些严重的问题。

一个示例场景—当开发一个应用程序时，它可能需要连接到一个数据库，或者需要一些证书，或者需要令牌(例如，Slack 令牌)来进行 API 调用。将这些密钥放在源代码中并不是一个好的做法，因为它们永远存在于源代码中，并且对所有能够访问源代码的人都是可见的。有时，开发人员将这些密钥保存在单独的本地文件中(添加到。gitignore)并使用不同的方法(如聊天、电子邮件等)共享这些密钥...这种方法仍然不好，因为秘密令牌是明文形式的，并且分散在团队成员之间。当一名团队成员转到另一个团队或另一个地方时，你如何处理这种情况？很难轮换密钥并与所有需要的团队成员再次共享它们。

你可以在[12factor.net](https://12factor.net/config)阅读更多关于为什么密钥应该与代码分开

# 解决办法

解决这个问题的方法是类似 [Hashicorp Vault](https://www.vaultproject.io/) 、 [Berglas](https://github.com/GoogleCloudPlatform/berglas) 、 [Google Secret Manager](https://cloud.google.com/secret-manager/) 、 [AWS Secret Manager](https://aws.amazon.com/secrets-manager/) 等秘密管理工具。其中一个流行的工具是 Hashicorp Vault，但是一个团队必须建立和管理他们自己的基础设施。

在本帖中，我们将了解 Google 托管秘密管理解决方案— [Secret Manager](https://cloud.google.com/secret-manager/) ，并了解存储和检索密钥和密码有多简单。

# 什么是秘密经理？

Google Secret Manager 是 Google Cloud Platform 的新成员，可以存储 API 密钥、密码、证书、敏感字符串等...你可以在这里阅读谷歌秘密管理器的发布说明[，在这里](https://cloud.google.com/secret-manager/docs/release-notes)阅读定价

Google Secret Manager 的访问控制与 Cloud IAM 集成在一起，项目所有者可以控制谁可以阅读这些秘密。

**roles/Secret manager . admin**:该角色提供管理秘密的完全访问权限。具有此角色的人员(或服务帐户)可以对机密执行 CRUD(创建、读取、更新和删除)操作。

**roles/Secret manager . Secret accessor**:该角色提供读取秘密的权限。具有此角色的人(或服务帐户)可以读取秘密值存储，如 API 密钥。

**roles/Secret manager . viewer**:该角色提供查看秘密元数据的权限。具有此角色的人员(或服务帐户)可以读取机密元数据，但不能读取存储的值。

现在，我们已经熟悉了 Google Secret Manager，让我们与这个服务进行交互。

# 与 Secret Manager 交互

# 先决条件

本帖假设如下:

1.  我们已经有了一个项目所有者角色的 GCP 项目。
2.  Google Cloud SDK ( `gcloud`)安装在您的工作站上。如果没有，那就参考我之前的博客-[Google Cloud SDK 入门](https://pbhadani.com/posts/getting-started-wth-google-cloud-sdk/)。

**注意**:您也可以使用 [Google Cloud Shell](https://cloud.google.com/shell/) 来运行`gcloud`命令。

# 启用秘密 API

为了与 Google Secret Manager 交互，我们需要启用这个 API。
运行以下命令来启用这个 API。

```
$ gcloud services enable secretmanager.googleapis.com --project=workshop-demo-23345
```

**输出**

```
Operation "operations/acf.xxxxxxf-xxxx-xxxx-xxxx-xxxxxxx" finished successfully.
```

# 创造一个秘密

运行下面的`gcloud`命令来创建一个秘密。

```
gcloud secrets create demo-secret --replication-policy="automatic"
```

**输出**

```
Created secret [demo-secret].
```

**注:**
1。这就产生了一个空的秘密，可以保存多个秘密版本(实际上是敏感信息)。
2。在这个例子中，我们使用`--replication-policy`作为`automatic`，但是也可以用`user-managed`来指定你想要放置秘密的位置。

# 向机密添加值

秘密版本包含实际的敏感值。

```
echo "my_super_secret_string" | gcloud secrets versions add 'demo-secret' --data-file=-
```

**输出**

```
Created version [1] of the secret [demo-secret].
```

**添加另一版本或更新秘密值**

```
echo "secret_update" | gcloud secrets versions add 'demo-secret' --data-file=-
```

**输出**

```
Created version [2] of the secret [demo-secret].
```

# 读取秘密值

我们可以访问最新的或特定的秘密版本

**获取最新秘密版本**

```
gcloud secrets versions access latest --secret='demo-secret'
```

**输出**访问特定秘密版本

```
gcloud secrets versions access 1 --secret='demo-secret'
```

**输出**

**注意:**切勿在应用程序中打印秘密值。

类似地，我们可以使用`gcloud`命令或应用程序中的客户端库在需要时获取秘密值，而不是放入源代码或某种属性文件中。

# 结论

Google Secret Manager 使开发人员能够以一种简单的方式存储和共享 API 密钥、密码等，而不必管理诸如 Vault 等工具

如果您有反馈或问题，请通过 [LinkedIn](https://linkedin.com/in/pradeepbhadani) 或 [Twitter](https://twitter.com/bhadanipradeep) 联系我

*原载于*[*https://pbhadani.com*](https://pbhadani.com/posts/google-secret-manager/)*。*