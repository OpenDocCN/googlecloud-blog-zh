# 跟踪与您的账单 id 相关联的 Google 云平台组织

> 原文：<https://medium.com/google-cloud/find-google-cloud-platform-organizations-associated-with-your-billing-ids-6b216618a592?source=collection_archive---------3----------------------->

## 鲁比·戈德堡遇见弗兰肯斯坦

您可能希望在多个 GCP [组织](https://cloud.google.com/resource-manager/docs/creating-managing-organization)中使用同一个账单账户；例如，SaaS 产品的多租户子域。

许多计费经理的下一个问题是如何跟踪在您的企业中百花齐放的情况，以及他们是否遵守政策。您可能已经从计费代码与单个项目的关联中猜到了，这很棘手。

您可以使用[计费 API](https://cloud.google.com/billing/reference/rest/v1/billingAccounts.projects/list) 列出与计费账户相关的项目；然后，您可以使用[资源管理器 API](https://cloud.google.com/resource-manager/reference/rest/v1/projects/getAncestry) 来获取项目的父组织。然而，有一点难以理解:访问资源管理器 API 的帐户需要项目所属组织的文件夹查看者权限。如果满足以下所有条件，这是可以避免的:

*   您对根计费组织拥有超级管理员权限。
*   计费用户权限最初被委派给该组织的成员。
*   受委托的用户在新组织中创建了一个项目。

你可能会问，这种魔力是如何发挥作用的？

![](img/6485e753e3daad8575b0f5956709af96.png)

布茨教授和自动操作餐巾纸(1931)。汤匙(A)被举到嘴边，拉动绳子(B ),从而猛拉勺子，将饼干(D)扔过鹦鹉(E)。鹦鹉在饼干和栖木倾斜后跳起来，把种子打翻在桶里。桶中的额外重量拉动绳子(I)，绳子打开并点燃打火机(J)，引发爆炸(K)，导致镰刀(L)切断绳子(M)，允许附有餐巾的钟摆来回摆动，从而擦拭下巴。

正如在[多租户 GCP SaaS 应用授权文章](/google-cloud/multi-tenant-google-cloud-platform-saas-applications-how-to-fbd16c8a9766)中所讨论的，一个适当授权的服务帐户可以代表一个域中的用户，即。通过[域范围的授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)模拟用户(忽略传统的 G 套件品牌；这也适用于云身份 API)。

在这种情况下，服务帐户将代表委托计费用户，使用资源管理器 API 获取其项目的父组织。

[该实用程序](https://github.com/demoforwork/public/tree/master/GCPOrgsAssociatedWithBillingIds)执行繁重的工作，并返回项目的父组织列表，其中一个查看者(或以上，例如所有者/编辑)已被授予与另一个组织关联的计费帐户的计费用户角色。

# 先决条件

## Python 和 Python 库

该实用程序依赖于 Python 和以下 GCP 和谷歌 API 库:

*   [googleapiclient](https://github.com/google/google-api-python-client/tree/master/googleapiclient)
*   google.oauth2

## 全域委托

从[云控制台](https://github.com/demoforwork/public/blob/master/GCPOrgsAssociatedWithBillingIds/cloud.google.com/console)，创建一个具有*全域委托权限*的服务账户，并下载 JSON 密匙。

*   从[IAM&admin->Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts)屏幕创建它(即。不是 API - >凭证屏幕)或全域委托权限将不可用
*   在生产中，您会希望将它存储在一个安全的存储中，比如 GCS，而不是您的文件系统；很容易不经意地将密钥和源代码一起上传，一旦上传到 GitHub 上，就很难删除旧版本的源代码。
*   注意客户端 ID:有一堆 ID…您想要的是从管理服务帐户->查看客户端 ID 屏幕…或者直接从下载的 JSON 文件中获取

从[云身份管理控制台](https://github.com/demoforwork/public/blob/master/GCPOrgsAssociatedWithBillingIds/admin.google.com)，以超级用户身份登录；超级用户角色是必需的，因为您可以通过这些设置向服务帐户授予完全管理 SDK 权限，否则将构成权限升级。

*   使用您之前记录的 id，[授权您在 GCP 创建的服务帐户](https://developers.google.com/admin-sdk/directory/v1/guides/delegation#delegate_domain-wide_authority_to_your_service_account)用于所需的范围，即【https://www . Google APIs . com/auth/cloudplatformorganizations . readonly、[https://www . Google APIs . com/auth/cloudplatformprojects . readonly](https://www.googleapis.com/auth/cloudplatformprojects.readonly)、[https://www.googleapis.com/auth/cloud-billing.readonly](https://www.googleapis.com/auth/cloud-billing.readonly)
*   从安全->高级设置->管理客户端 API 访问窗口执行此操作

# 安装

下载[文件](https://github.com/demoforwork/public/tree/master/GCPOrgsAssociatedWithBillingIds)。

在脚本中配置所需的域、组织和 JSON 密钥文件

运行脚本:

```
$ python list_orgs_associated_with_billing_ids.py
```

# 承认

非常感谢 Cyrus Harvesf 的关键见解，即二级域名上项目的计费用户通常是一级组织的成员，因此可以通过一级域名上的域名范围委托来进行跟踪；也谢谢你在我迷失在荆棘丛中时指引我回到正道。