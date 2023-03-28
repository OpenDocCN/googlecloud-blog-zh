# GCP Onboarding -使用 Azure AD 的云身份用户供应和单点登录

> 原文：<https://medium.com/google-cloud/gcp-onboarding-cloud-identity-user-provisioning-and-sso-using-azure-ad-4b6d0ec5a691?source=collection_archive---------3----------------------->

这是一个自以为是的指南，旨在利用您现有的身份提供者(AzureAD)来设置您组织的 Google 云平台(GCP)。

![](img/7fe5323c95c2416c00d119b43bb3363d.png)

迈克尔·泽兹奇在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上的照片

## 什么是云身份？

云身份是一个身份即服务(IDaaS)解决方案，允许您集中管理可以访问谷歌云平台(GCP)的用户和组。这是您的组织进入 GCP 的先决条件。我们使用云身份对用户进行身份验证(AuthN)。

## Azure AD 是什么？

Azure AD 是基于云的身份和访问管理服务。对于我们今天的上下文，Azure AD 将被用作外部身份提供商(IdP)和身份源，您的组织将使用这些身份来加入 GCP。

## 为什么？

如果您的组织使用 Azure AD 等现有身份提供商，使用此外部 IdP 的用户配置和单点登录(SSO)将实现更快的 GCP 入职，并使您的管理员的工作更轻松，中断更少。

## 怎么会？

我们将在 Google 管理控制台上创建一个用户，并在 Azure 上设置一个企业应用程序，使用该用户的凭据进行授权。这将使 Azure AD 能够自动配置用户。

对于 SSO，我们将从管理控制台下载证书并上传到 Azure，然后配置 SAML 和登录端点。

# 把手放在某物或者某人身上

## 准备云身份

首先要做的是设置一个用户帐户，该帐户将用于从 Azure AD 提供用户。我们将为该用户创建一个组织单位(OU ),并为该用户分配权限，以便能够管理云身份上的用户和组。要执行这些步骤，您应该拥有超级管理员权限。

1.  在云身份管理控制台上，在*目录- >组织单位*([https://admin.google.com/ac/orgunits](https://admin.google.com/ac/orgunits))下，创建一个名为“自动化”的新 OU

![](img/2d3bdf1cf80f3a553b4bca447707c0a4.png)

创建新的 OU

2.在*目录- >用户*([https://admin.google.com/ac/users](https://admin.google.com/ac/users))下，创建一个新用户，该用户将用于用户供应。

![](img/848be55d2535192b13b8dcb14b140a15.png)

创建新用户

3.在*帐户- >管理角色*([https://admin.google.com/ac/roles](https://admin.google.com/ac/roles))下，创建一个新角色并分配权限，如下所示

![](img/1611d8a1903a61e6b212b9a190efc926.png)

创建角色

![](img/5dd7149523e69444d0cb8cf17221035d.png)

为角色分配权限

确保将角色分配给我们之前设置的用户。我们现在可以继续在 Azure AD 上设置用户配置了。

## 设置用户设置

1.  在 Azure 门户([https://portal.azure.com/](https://portal.azure.com/))，在 *Azure Active Directory - >企业应用*下，通过搜索“Google Cloud”并点击结果列表中的“**Google Cloud/G Suite Connector by Microsoft**”项，创建一个新的应用。将其命名为“Google Cloud (Provisioning)”。

![](img/56c3627e6dad2e6fe3f72520b802bc50.png)

安装谷歌云连接器

2.在该应用程序的*管理- >属性*下，

*   将**设置为启用，以便用户登录**为否
*   将**需要用户分配**设置为否
*   将**对用户可见**设置为否
*   点击**保存**。

![](img/3ea41e78f37f502705e6c9e96ffd64e0.png)

设置供应

3.在该应用程序的*管理- >供应*下，

*   将**设置为启用，以便用户登录**为否
*   将**需要用户分配**设置为否
*   将**对用户可见**设置为否
*   点击**保存**。

![](img/d138dbc81f8893164eb91306cbd7bb9b.png)

授权用户

![](img/2c18d3610178aa92ecd1b3564ee7dc6e.png)

管理员凭据

在*映射*下，点击“userPrincipalName”并将“源属性”设置为“邮件”。

![](img/d30fd09283bc2798c178a525ceee66e9.png)

更新映射下的源属性

现在，如果为空(可选)字段，用“_”作为*默认值，更新“姓氏”和“给定姓名”属性。保存这些更改。*

根据您的广告许可证，您可以在*管理- >用户和群组*下分配用户和群组。一旦您在*管理- >供应*下开始供应，这些用户和组将在 Google Cloud 上自动供应。

![](img/436f0b127b98986089d2c1cf96265e59.png)

开始设置

您应该很快就会在 Cloud Identity 上看到分配的用户和组。对于故障排除，您可以使用*查看配置日志*选项。

## 设置单点登录

对于 SSO，让我们使用不同的名称再次部署相同的企业应用程序。为了清楚起见，我们使用了同一个连接器应用程序的两个实例(来自 Azure AD Gallery 的企业应用程序)。一个用于以前的配置，一个用于现在的 SSO。

![](img/21fbaf58de2face321a0d0ac5342c056.png)

安装 Google Cloud Connector(这次是为了 SSO)

在该应用程序的*管理- >属性*下，

*   将**设置为启用，以便用户登录**为是。
*   如果您希望所有用户使用 SSO，请将**需要用户分配**设置为否；如果您希望选择选择用户或组，请设置为是。
*   如果您在上一步中配置了所需的用户分配，请在*管理- >用户和组*下分配您想要使用的用户或组。

![](img/400c42efb7df18ed98253bb4ba02f606.png)

设置 SSO

在*管理- >单点登录*下，选择 *SAML* ，编辑*基本 SAML* 的配置如下:

1.  输入回复网址为[*https://www.google.com*](https://www.google.com)
2.  登录网址为[https://www.google.com/a/YOURDOMAIN.COM/ServiceLogin？continue = https://console . cloud . Google . com/](https://www.google.com/a/YOURDOMAIN.COM/ServiceLogin?continue=https://console.cloud.google.com/)

![](img/d513c000d51214631690c05be9ebbe30.png)

配置基本 SAML

点击*保存*并点击 *X* 关闭对话框。在 *SAML Certificates* 卡上，下载 Base64 证书并将其保存到您的本地机器。

![](img/bfd418d45e6e7232998daea5ff30913d.png)

下载 Base 64 证书

从设置的 Google Cloud 卡中复制登录 URL。

![](img/f70cf251d74da825210eda1f14e62686.png)

复制登录 URL

在*属性的&权利要求*卡片上点击编辑并删除附加权利要求。

![](img/ddebdd1cffc11ec7721c09c04e62be01.png)

在删除附加声明之前

![](img/06e618fb87ab020f0d8d248333dc69c0.png)

在删除了附加声明之后

现在，让我们在云身份管理控制台上设置 SSO。在*安全- >认证- >与第三方 IdP* 的 SSO 下，点击*添加 SSO 配置文件*。使用 Azure 上 SSO 应用程序的登录和注销 URL 值作为登录和注销 URL。上传之前下载到本地机器的证书并保存。

![](img/86f91ce52c25156d1da969d2eed5b4ed.png)

在管理控制台上设置 SSO 配置文件

现在，最小化该卡，转到同一页面上的*管理 SSO 配置文件分配*卡，并为自动化 OU 禁用 SSO。确保您是专门为自动化 ou(在左侧您的组织名称下选择它)这样做的。

![](img/68cb58441ffce8bb577711d5c81706fa.png)

从 SSO 中排除自动化 OU

你已经准备好在 https://console.cloud.google.com/测试单点登录了。出现提示时，在 Google 登录页面上提供电子邮件，您将被重定向到带有 SSO Microsoft 徽标的 Azure AD 登录。

## 最佳实践/陷阱

对于大型组织来说，只有 IT 团队可以使用 GCP 平台。因此，建议将用户配给和 SSO 企业应用程序分配给一组特定的用户组。

如果您使用免费版的 Cloud Identity，并且您将调配超过 50 个用户，则您必须请求增加配额。

## 结论

我们在 Google Cloud 上使用您现有的外部 IdP 设置了用户供应和单点登录，以便您可以让您的员工更快地到达 GCP，享受 Google Cloud 体验！感谢阅读。

## 参考

[](https://cloud.google.com/architecture/identity/federating-gcp-with-azure-ad-configuring-provisioning-and-single-sign-on) [## Azure AD 用户配置和单点登录|身份和访问管理|谷歌云

### 若要让 Azure AD 访问您的云身份或 Google Workspace 帐户，您必须在您的中为 Azure AD 创建一个用户…

cloud.google.com](https://cloud.google.com/architecture/identity/federating-gcp-with-azure-ad-configuring-provisioning-and-single-sign-on) [](https://cloud.google.com/architecture/identity/federating-gcp-with-azure-active-directory) [## 将 Google Cloud 与 Azure Active Directory 联合起来|身份和访问管理

### 本文描述了如何配置云身份或 Google Workspace，以使用 Azure AD 作为 IdP 和源…

cloud.google.com](https://cloud.google.com/architecture/identity/federating-gcp-with-azure-active-directory) 

## 进一步阅读

[](https://learn.microsoft.com/en-us/azure/active-directory/app-provisioning/how-provisioning-works) [## 了解如何在 Azure Active Directory-Microsoft Entra 中提供应用程序

### 自动配置是指在用户需要访问的云应用程序中创建用户身份和角色…

learn.microsoft.com](https://learn.microsoft.com/en-us/azure/active-directory/app-provisioning/how-provisioning-works)