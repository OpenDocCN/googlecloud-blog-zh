# 用 Google 云服务帐户模拟用户

> 原文：<https://medium.com/google-cloud/impersonating-users-with-google-cloud-platform-service-accounts-ba762db09092?source=collection_archive---------0----------------------->

## 能力越大，责任越大

正如 Sal Rashid 在他的文章中描述的那样，IAM [serviceAccountActor 角色](https://cloud.google.com/iam/docs/service-accounts#the_service_account_actor_role)使另一个用户或服务帐户能够模拟一个服务帐户(这个角色现在已经被 [serviceAccountUser 角色](https://cloud.google.com/iam/docs/service-accounts#the_service_account_user_role)取代)。

但如果你想做相反的事情，即。用服务帐户模拟用户？这是基于服务的架构中的一个常见用例:服务将代表用户执行操作(例如，存储记录、检索 blob)。

首先，简单介绍一下为什么您可能想要模拟用户，而不是仅仅使用服务帐户进行授权。后者很简单，但是违反了最小特权(就像构建一个只使用 [API 键](https://cloud.google.com/docs/authentication/api-keys)的移动应用程序),并且如果用户也可以直接访问(例如，从 Google 云存储上传/下载对象，或者在 BigQuery 上运行查询)服务与之交互的 Google 云平台管理的资源，那么它就会崩溃:显然，基于服务帐户的权限和直接用户访问之间存在不匹配。

您可以通过显式设置基于用户的权限来解决这种不匹配(Google Cloud Storage object ACLs 除外，它在一定程度上基于上传对象的用户)，但这会增加处理和管理开销。这就把我们带到了用户模拟。

# 用户模拟

您可以让服务帐户模拟托管用户，即。通过[全域授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)代表云身份行事(忽略传统 G 套件品牌；这也适用于云身份 API)。这是我们将在本文中重点讨论的方法。

这解决了上面提到的基于服务帐户的授权的几个问题:

*   您的应用程序创建的资产需要向最终用户授予 ACL。您可以通过 IAM APIs 手动管理它，但是用户模拟会自动处理它。
*   Google 云存储对象 ACL 部分基于上传对象的用户；用户模拟确保这些 ACL 反映的是用户而不是服务帐户。
*   您的应用程序需要根据客户的用户的身份向他们公开适当的资产。手动管理将需要您在应用程序中构建一个 ACL 层；用户模拟使它自动发生。

## 限制

那么，既然空气中弥漫着这种魔力，你为什么不这样做，祝贺你自己出色地完成了工作，然后回家呢？有几个原因:

**最低特权**

正如您可能已经从“域范围”的命名中猜到的那样，用户模拟的范围可以缩小到 GCP API 方法，但不能缩小到域内的用户或资源。这意味着服务帐户可以访问其方法范围内的所有资源(例如，读取 Google 云存储)，因此最小特权问题仍然存在。

*   如果你在管理自己的用户和数据，这还不错，虽然不是很好。
*   如果您正在构建需要访问客户拥有的资源的 SaaS 应用程序，情况会更糟，因为它们的许多用户资产对客户来说是机密的。不应与您的应用程序共享。

**审核**

就 Google 云平台而言，一旦建立了身份验证，实际用户就在创建/访问资源。这意味着审计跟踪、计费、配额等。都链接到用户，而不是您的应用程序或其服务帐户。

这值得特别注意，因为大范围的用户模拟需要强大的补偿控制；强大的治理流程和自动化会有所帮助。

## 减轻

[VPC 服务控制](https://cloud.google.com/vpc-service-controls/)(私下测试版)可以帮助减轻最小特权问题，尽管它们确实会招致管理间接税。

它们使您能够围绕 Google 云平台资源(如云存储桶)定义基于网络的安全边界，以将数据限制在虚拟私有云(VPC)内，并帮助降低数据泄露风险。

您和您的客户可以将资源放在安全区域，并建立基于 IP 的访问限制，这样机密信息就不会与您的应用程序共享。

有一个关键的警告:VPC 服务控制是失效开放的，而不是失效关闭的，所以没有明确在 VPC 安全区的数据将可以跨越 VPC 的边界。强大的治理流程和自动化有助于降低这种风险。

# 代码示例

这里有几个 Python 示例可以帮助您入门…

创建具有任意用户所有权的 GCS 存储桶

上传具有任意用户所有权的 GCS 对象

![](img/5e95b5ce6591be38aacef9d9bc113a2d.png)

该作品已经被作者发布到公共领域

# 下一步是什么

阅读下面的内容，了解更多关于本文的基本概念:

*   [全域授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)。
*   [VPC 服务控制](https://cloud.google.com/vpc-service-controls/)。

阅读以下指南，了解如何实施:

*   [谷歌云平台 API](/google-cloud/faster-serviceaccount-authentication-for-google-cloud-platform-apis-f1355abc14b2)更快的服务账户认证。
*   [服务帐户模拟](/google-cloud/using-serviceaccountactor-iam-role-for-account-impersonation-on-google-cloud-platform-a9e7118480ed)(注意 IAM serviceAccountActor 角色已被 serviceAccountUser 角色取代)。
*   [多租户 B2B SaaS 认证和授权](/@fargyle/multi-tenant-google-cloud-platform-saas-applications-how-to-fbd16c8a9766)。