# 谷歌计算引擎元数据服务

> 原文：<https://medium.com/google-cloud/google-compute-engine-metadata-service-de9d71ea44e0?source=collection_archive---------0----------------------->

在构建和部署管道中管理 API 密钥和其他秘密可能是一件痛苦的事情。例如，在生产中，您希望使用生产信用卡网关，并将您的生产凭证用于 New Relic。在 staging 中，您希望使用测试网关，并为 New Relic 使用一组不同的凭证。开发和演示服务器有更多不同的配置。记录所有这些可能是一件痛苦的事。

我已经用 Chef 脚本和数据包处理了这种复杂性，但即使这样，我有时也会搞砸。此外，如果一个 API 键需要更新，我们将针对服务器重新运行 chef cook，这有时需要关闭服务器。这种解决方案是有效的，但它似乎是不雅的。自从我开始在谷歌工作以来，我一直试图找出是否有更好的方法来管理秘密，至少在 GCP 是这样。

每个 Google 计算引擎实例都可以访问存储在元数据服务器上的元数据。元数据存储为键:值对，位于特定的 URL。每个实例都有一些默认的元数据(主机名、启动和关闭脚本以及实例 ID)。用户还可以指定实例可以访问的自定义元数据。启动脚本或应用程序可以检索这些信息来设置环境变量、连接到外部服务以及配置实例。

元数据是在实例或项目级别设置的。从同一模板(如自动缩放的托管实例组)创建的实例将具有相同的自定义元数据。可以使用`gcloud`，通过特定语言的 API，或者使用 web 控制台来设置和查看元数据。您还可以更新元数据，该值几乎立即传播到所有实例。

# 结构化您的数据

元数据服务是管理配置和机密的其他方法的替代方法。它支持单值和目录结构。组织数据的一种可能方式是为每个环境(登台、生产、演示)创建一个目录，然后在每个目录中创建一组相同的键(具有特定于环境的值)。特定于实例的元数据可以用来确定实例应该访问哪组项目范围的键。这只是一个建议。有许多其他的方法可以很好地为您的项目构建数据。

# 访问数据

从一个正在运行的实例中，通过向`http://metadata.google.internal/computeMetadata/v1/`发出请求来访问元数据服务器。您需要在请求中包含“Metadata-Flavor: Google”头。下面是一个使用 curl 的示例:

```
curl -H "Metadata-Flavor: Google" [http://metadata.google.internal/computeMetadata/v1/](http://metadata.google.internal/computeMetadata/v1/)
```

这里有一个在 Ruby 中使用`Net/HTTP`的例子。

# 从 Web 设置和查看

您可以在[https://console.cloud.google.com/compute/metadata](https://console.cloud.google.com/compute/metadata)的网站上查看、设置和更新项目范围的元数据。您可以从“计算引擎”下的“实例”页面更新实例元数据。[https://console.cloud.google.com/compute/instances](https://console.cloud.google.com/compute/instances)

# 处理变更

根据我的经验，改变一个 API 键或另一个配置选项通常需要重启这个过程。有时，变更甚至需要调配全新的虚拟机。当您的系统能够在不影响客户的情况下处理时，要求重启是可以的，但这并不总是现实的。正如我们所希望的那样，粘性会话和保持连接打开(就像 web 套接字一样)在 web 应用程序中很常见。

元数据服务器具有等待更改功能，因此当值发生更改时，您的应用程序会收到通知，并且无需重新启动即可做出响应。关于等待更新和使用 ETags 的更多信息可以在[文档](https://cloud.google.com/compute/docs/storing-retrieving-metadata?hl=en_US&_ga=1.112327874.711715654.1473891887#waitforchange)中找到，但是我将在这里解释基础知识。

要使用 wait-for-change 特性，您需要发出一个普通的请求，并将`?wait_for_change=true`追加到查询字符串中。当键值改变时，请求返回新值。这里有一个例子。

```
curl -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/project/attributes/my_key?wait_for_change=true"
```

# 限制

元数据服务有一些限制。当涉及到存储机密时，最大的问题是元数据对于在您的项目中通过验证的任何人或机器都是明文可见的。在大多数情况下，这是好的，但这可能是一些应用程序的问题。

此外，自定义元数据被限制为 32，768 字节的 key:value 数据。这意味着它不是存储启动脚本、大型 JSON blobs 或序列化对象的好解决方案。如果你需要比元数据服务器允许的更多的空间，你可以使用谷歌云存储来存储信息。然后，您可以将存储桶的路径存储在元数据服务中。

元数据服务还有其他几个有用的特性，我无法在这里介绍，包括维护通知。要了解更多关于元数据服务和与它交互的不同方式，请查看[文档](https://cloud.google.com/compute/docs/storing-retrieving-metadata)。

10/20/16

*原载于 2016 年 10 月 20 日 www.thagomizer.com*[](http://www.thagomizer.com/blog/2016/10/20/metadata-service.html)**。**