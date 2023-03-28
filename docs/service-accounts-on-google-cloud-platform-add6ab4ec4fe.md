# 谷歌云平台上的服务账户

> 原文：<https://medium.com/google-cloud/service-accounts-on-google-cloud-platform-add6ab4ec4fe?source=collection_archive---------1----------------------->

当我与客户一起工作时，经常会发生这样的情况，他们没有太多使用谷歌云平台的经验，即他们通常会注册和创建项目，将开发人员添加到项目中，但例如，如果他们想要给开发人员细粒度的访问权限，设置服务帐户就不那么简单了。所以在这篇文章中，我想给出一个关于服务账户的概述，这是谷歌云平台开发中经常使用的功能，它是为非开发人员编写的。

# 什么是服务帐户？

服务帐户类似于普通用户帐户，不同之处在于通常在访问 GCP 产品的代码中使用。正如用户可以有不同的角色一样，服务帐户也是如此。用于验证 GCP 的数据作为密钥(实际上是 json 或 p12 格式的文件)提供，然后可以在代码中使用。因此，当用户通过电子邮件和密码进行身份验证时，服务帐户通过密钥文件进行身份验证。因为它是文件，可以很容易地复制，共享等，你的项目的安全性可能会受到影响，所以要小心你的服务帐户文件存储在哪里，你如何共享它们，以及与你共享它们的人在哪里使用它们，我们是他们保存它们。

服务帐户管理属于 IAM & admin 部分

![](img/fdb5851f400a3d06c753490caaf74e7a.png)

# 如何设置服务帐户

这是服务帐户列表，如下图所示，有 2 个服务帐户，我们将创建一个新帐户，为此，请单击“创建服务帐户”按钮。

![](img/db0d63d3f3cab69164ec30b04bfca6df.png)

在弹出窗口中，有一些东西需要设置:

-服务帐户名称-这只是将显示的名称，因此请为其提供一些有意义的描述，尤其是如果您有多个帐户要轻松导航的话。在这种情况下，我假设想要创建一些脚本或服务，它们将在服务器上运行，并从第三方 API 获取数据，上传到云存储，然后上传到 BigQuery。

-服务帐户 ID-最初这是从服务帐户名称派生的，但如果您愿意，可以更改，从技术上讲，这并不重要。

![](img/d105aecca3d91b8570fa3f35d3b39f3e.png)

-角色-您可以在此为该服务帐户定义多个角色。在这个例子中，我将为 BigQuery 和云存储添加角色。自然，所有这些角色可能都不容易理解，我想最好的资源是这篇综合文章，其中解释了所有可能的角色[https://cloud.google.com/iam/docs/understanding-roles](https://cloud.google.com/iam/docs/understanding-roles)。

当您点击选择一个角色字段时，将弹出一个小窗口，显示产品及其角色的列表。

![](img/617e8b7db211d7be67e88e1e242415be.png)![](img/99819f6fef835e18ce4ebd8cbf205ab8.png)

最后一件事是检查选项提供一个新的私钥，当你点击保存按钮，json 文件将被下载。

![](img/a53ae7c1ed67c6e37d0178e0578d2487.png)

创建后的消息。

![](img/1ed0f0f82ab1a2c39fad1d88bfea7204.png)

这就是服务帐户文件密钥的内容。

```
{
  "type": "service_account",
  "project_id": "just-some-project",
  "private_key_id": "e5703451d59c7f1152517122a9e8824",
  "private_key": "-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCgGPVstF25mezs
SrIQ+jW0neRqEaCi3800d5Sm5F36cq2vTl9kFkahz4zmQiyNqgyBayQbipHJxVV0
feI65nh+BXj4nX2ZC1vLGRIlcRwzauW/58wcMAtaTPRstXwk/LkJJURG1klSF+kD
p3bD6VzpE5nJt396jy0g7q/Mlh/gGNzTS05BsWHl4+iurtlqCzB5XKiKpDlF9CC8
sa/tAvex1mCuUgjZKcMMpqSOuVVFQeyk9xx6TO3GbQfYvHPwpZBrEY14Aplw2Ikg
4qiQHBVSBACA9LHRTRHYGmwvrAYx7sWQk8WI9Q/oN5xzZOp5rrWJGl1XZxlryLKL
tOTQPCFPAgMBAAECggEAB4t1kXRkkFkdLbuNbbhk7IieWZNQFyzl3kmeY0kS1rkt
zzupyi00ke6WjMlkkR6I7VZvwq6ClVgr4UwHa9AtsH6wh0LgxuQepNEfJk2QzymD
/0m8uD1ZY52JzU+2NIYGcFRlxdLWbLx8ujPbqHroaoGGd1ei9dMmhM1K0qDmiqh+
L6xhEoujxySgYobDk5ZZt3EbG1N6N4D+C+/6mguakXKU2nl7kyEGngykOij67L+C
nnK/4R4HHUNU5YOTriq+7WYNwMOzHrVT5z41+0yG3FJvIZ0Iy57x5RNw5xVHoYNn
iX8m7jLQ/ByHCeJ9HcqUdIj3ovkvtTS2zsozaCiVHQKBgQDVsKwBp8KbgaC9xluY
NuN88t/m6SPvBn9k4xaLnvQkjCO/LKOlRsxSPzmxzTc7lOocVXsr7IyzoDH4QMpo
yjbajjqvcUG2L2oE1iG9nKLtTC2iaeR/OPJRRagvPhgPe8cskqy/I60tlfvgr7kB
zRftCcJXTI3J94gFzd0tnEB9awKBgQC/y9U9MLrm0aQk9+VWJDMn9X0/6qSXaxRK
DOlijUjWSvGPOZ+hetueDZnZyBSjuFjcX8hV2qMn2TDPnd/KxGxhuB8Xe6ExTWnT
B6SHT3BpG6+4u9atuoSUY6AYGbFCP2v7jf2FTPXoAn
C2Z+XuND4t7wlxxIDxRzaBx95wwmEkrm0A2uNcpy1z2kPYGKS/U8blnGC8ie7anT
g89mor9s2Vkno6rJ+r0nOtzy
-----END PRIVATE KEY-----
",
  "client_email": "data-ingestion-pipeline@756.iam.gserviceaccount.com",
  "client_id": "109097848740185",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://accounts.google.com/o/oauth2/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/data-ingestion-pipeline%40r756.iam.gserviceaccount.com"
}
```

正如我前面所写的，下载的文件在客户端库中用于访问 GCP 服务。当然，最重要的是保证该文件的安全，因为任何获得该文件的人都可以根据授予该服务帐户的权限和角色访问您的 GCP 产品或访问数据。

# 服务帐户的常见操作

在服务帐户列表中。

![](img/9f343b678c4c1708409966911e17a9fc.png)

当你点击设置，你可以选择编辑，删除和创建密钥

![](img/05e28683af46ddb0bfeb7eaa539cd527.png)

**编辑** —为您提供更改服务帐户描述的可能性

**删除** —删除服务账号

**创建密钥** —这提供了新的密钥/文件。这意味着一个服务帐户可以有多个密钥/文件。点击垃圾桶，你可以删除它们。如果您认为密钥/文件已经泄露，这是您应该做的第一件事。

其他常见的事情是修改为服务帐户分配的角色。

当您转到 IAM 部分时，您将看到帐户列表，包括用户和服务帐户。现在，根据屏幕分辨率，您可能需要向右移动底部的滑块，以进入选项:)。

![](img/40a5a9c8c95390688226aae4d443b933.png)

您可以看到分配的角色以及编辑或删除角色的选项。

![](img/53890aec4a55fda21f62e5dfde8b4df7.png)

当您点击笔(编辑)时，您可以删除、更改或添加与创建时类似的角色。

![](img/a4a097ed55025f5803a1aa829e779955.png)

我想在此介绍的服务帐户的最后一个功能是分配对服务帐户的访问权限。基本上，当您创建服务帐户时，您可以为一个或多个用户帐户分配访问权限。使用此功能，您可以有选择地授予开发人员对服务帐户的访问权限，而无需授予他们对所有帐户的一般访问权限。当他们有权访问服务帐户时，他们可以修改它、更改角色等。

要将用户分配到服务帐户，请转到服务帐户列表，选中您要分配访问权限服务帐户，然后单击右上角显示信息面板。

![](img/17206873f7c58ae8e69d5ef5cba849ca.png)

在分配用户的弹出窗口中，您需要输入用户的电子邮件(您还需要单击它)并选择一个或多个角色。这里只有 3 组角色:项目、服务帐户和 IAM，选择后，它们将仅应用于此服务帐户，而不是整个项目。

![](img/b45d3a8218ba3e49efa164cf0b5ffe2f.png)

添加后，您可以在角色下看到分组的成员。

![](img/f4fecf7870186c9c9eff8197451a1a2a.png)

当您切换“显示继承的权限”时，您还可以看到继承的权限，即具有隐式访问此服务帐户的角色的成员。

![](img/3ffbf625a04211f1a57286088584e967.png)

我希望这篇文章能让你更清楚地了解什么是服务账户以及如何处理它们。

我正在策划关于谷歌云平台[https://www.gcpweekly.com/](https://www.gcpweekly.com/)的每周时事通讯。如果您想在每周一收到与 GCP 相关的新闻和文章，请订阅。