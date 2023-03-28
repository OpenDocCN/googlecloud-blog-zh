# 多租户谷歌云平台 B2B SaaS 应用

> 原文：<https://medium.com/google-cloud/multi-tenant-google-cloud-platform-saas-applications-f40cd9ae72a0?source=collection_archive---------3----------------------->

## 概念

# 概观

谷歌云平台的许多客户和合作伙伴在 GCP 上构建 B2B 应用；其中一些应用程序可以在其他平台上使用，无论是在内部还是在云中(例如 [SAP](https://cloud.google.com/sap/) )，而一些应用程序则更好地利用 GCP 的托管服务(例如 [JDA](https://cloudplatform.googleblog.com/2015/06/JDA-embraces-new-ways-of-thinking-with-Google-Cloud-Platform.html) 、 [EnergyWorx](https://cloud.google.com/customers/energyworx/) 、 [Leanplum](https://cloud.google.com/customers/leanplum/) )在 GCP 上实现软件即服务(SaaS)。

后一类企业面临三个有趣的挑战:

*   管理客户用户的身份验证和授权。
*   解决客户对隔离的担忧。
*   为您的客户提供所有服务的整体界面。

我们将在操作指南中探讨应对这些挑战的方法，但首先需要更多的背景知识。

# 认证和授权

谷歌云提供健壮的[认证](https://developers.google.com/identity/sign-in/web/sign-in)和[授权服务](https://cloud.google.com/iam/)；挑战在于调和两个世界:

*   用户身份可能不是[谷歌云身份](https://cloud.google.com/identity/)或其他[支持的云原生身份](https://firebase.google.com/docs/auth/)；典型的企业身份由[活动目录](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)管理。
*   谷歌云平台的用户认证和授权基础设施基于谷歌账户(例如[谷歌云身份](https://cloud.google.com/identity/))。

在许多情况下，这可以通过[服务账户](https://cloud.google.com/iam/docs/understanding-service-accounts)来弥补；如果你希望将 GCP 的一些托管服务直接展示给客户的最终用户，比如将文件上传到[谷歌云存储](https://cloud.google.com/storage/)，在 [BigQuery](https://cloud.google.com/bigquery/) 上运行分析，这就更加困难了。

# 隔离

除了 [ACL](https://cloud.google.com/iam/docs/overview) 和[基于网络的控制](https://cloud.google.com/vpc-service-controls/)之外，谷歌云平台提供了许多隔离边界，包括项目，现在还有文件夹和组织。

关键是以符合客户期望的方式应用[谷歌云的深度防御](https://cloud.google.com/security/overview/whitepaper)方法，客户的期望通常植根于外壳坚硬、内部柔软的内部世界。

# 整体界面

当人们想到 SaaS 应用程序时，他们通常会想到 UI。然而，您的许多客户也需要能够与您的 SaaS 服务交互的编程方式(例如 [Google Drive API](https://developers.google.com/drive/) 、 [Google Apps 脚本](https://developers.google.com/apps-script/))。

Google Cloud Endpoints 提供了一种开放的 API 兼容方式来公开您的 GCP 服务；然而，你们中的许多人要么将一些服务保留在本地，要么随着时间的推移迁移到 GCP，并且需要为所有应用程序(包括“遗留”应用程序)提供一个整体接口。

![](img/8b6b63b0dc6c6801ce0cb77b54d05820.png)

[根据知识共享协议 CC0](https://pxhere.com/en/photo/861070) 免费发布

# 下一步是什么

阅读[企业组织的最佳实践](https://cloud.google.com/docs/enterprise/best-practices-for-enterprise-organizations)以了解更多关于支持本文的最佳实践。

参考谷歌云[信任&安全](https://cloud.google.com/security/)和谷歌云平台[安全概述](https://cloud.google.com/security/overview/)微型网站，了解更多关于谷歌云安全的信息。

阅读以下指南，了解如何实施:

*   [多租户 B2B SaaS 认证和授权](/@fargyle/multi-tenant-google-cloud-platform-saas-applications-how-to-fbd16c8a9766)。
*   [多租户 B2B SaaS 隔离](/@fargyle/multi-tenant-google-cloud-platform-saas-applications-how-to-3fc48162ef39)。