# 多租户谷歌云平台 B2B SaaS 应用指南

> 原文：<https://medium.com/google-cloud/multi-tenant-google-cloud-platform-saas-applications-how-to-3fc48162ef39?source=collection_archive---------1----------------------->

## 隔离

# 概观

本文解决了在[概念文章](https://medium.com/p/f40cd9ae72a0/edit)中描述的隔离挑战。

概括来说，谷歌云平台提供了许多隔离边界，包括项目，现在是文件夹和组织，此外还有 ACL 和[基于网络的控制](https://cloud.google.com/vpc-service-controls/)。

[文档](https://cloud.google.com/iam/docs/overview)很好地概述了 ACL、项目、文件夹和组织策略。

在本文中，我们将重点关注可伸缩的基于项目的隔离、组织和域之间的关系、基于网络的控制以及它们对多租户 SaaS 应用程序的影响。

除了这里描述的隔离约束之外，您选择将应用程序和数据服务分配给项目和/或域的方式将取决于主权和延迟约束。

# 项目

项目是最初的谷歌云平台隔离边界；这也是谷歌对与组织无关的客户使用的界限。

如果此边界解决了您的客户感知的安全问题，并且您不需要支持多个 SAML IDPs(请参见身份验证和授权文章)，我们建议使用此边界，因为它支持有用的合资企业和数据共享结构，如 VPC 服务控制安全区域(请参见下文)。

基于项目的隔离的一个自然问题是项目的扩散。租赁单元有助于缓解这种担忧。

## 租赁单位

[租赁单元](https://cloud.google.com/service-infrastructure/docs/tenancy-units-tutorial)提供基于服务、基于消费者的隔离环境。当新客户开始使用您的服务时，您可以在一个租赁单元中创建特定于该客户的所有资源。

您可以在一个租赁单元内为每个客户创建一个单独的单租户应用程序实例，而不是构建一个本地多租户托管服务。这样，您就创建了一个“多单租户”服务:从消费者的角度来看是多租户的服务，但实际上是作为单独的单租户应用程序实现的。

这最适用于按客户隔离数据或按用户隔离应用程序基础架构的应用程序(即不是基于 App Engine 或 Google Kubernetes 引擎的完整多租户 PaaS 服务)。

# 组织和域

首先，一点术语:组织！=组织。

*   [云平台组织资源](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#organizations)代表一个组织(例如一家公司)，是 GCP 资源层次结构和 IAM 策略中的根节点。它与云身份域相关联，云身份域代表组织中所有 Google 帐户的虚拟组，并公开基于域的策略和安全控制，如 SAML SSO 和[管理控制台组](https://support.google.com/a/answer/33329)。
*   云身份域还包括一个名为[组织](https://support.google.com/a/answer/4352075?hl=en)的资源(忽略传统的 G 套件品牌；这也适用于云身份)；这是用来组织用户和应用不同种类的政策给他们:他们可以访问哪些谷歌产品(如套件，YouTube，云控制台)，谁可以管理他们(如他们的密码管理员)。

我们将主要关注云平台组织资源及其相关的云身份域。从 Google 云平台组织策略的角度来看，子域相当于域；然而，从云身份管理的角度来看，情况并非如此。

事实上，云身份域是用户管理和 SAML SSO 的基础，如果您为客户提供终端用户管理功能，这将带来许多影响:

*   域代表了一个非常强大的隔离边界:这也是 Google 为其企业客户使用的边界。它们也是您的客户熟悉的一个界限；虽然组、文件夹和项目也是隔离边界，但它们需要更多的解释。
*   您需要为希望启用 SAML SSO 联合的每个 IDP 分配一个单独的域或子域。正如在身份验证和授权文章中所讨论的，可以通过使用 SAML 代理来减轻这种约束。

除了上面讨论的 SAML 联合含义之外，在选择多客户域还是单客户域时，还有许多因素需要考虑。

## 身份表征

客户用户和为每个客户分配的域上的身份之间的映射是直观的(例如[userid@customerdomain.mydomain.com](http://chevron.slb.com/))。

在单个域代表所有客户的情况下，您需要在身份中包含客户信息(例如[userid_customerdomain@mydomain.com](http://chevron.slb.com/))。如果你希望将 GCP 的一些托管服务直接暴露给客户的最终用户，例如，将文件上传到[谷歌云存储](https://cloud.google.com/storage/)，在 [BigQuery](https://cloud.google.com/bigquery/) 上运行分析，而不使用 SAML 联邦，这尤其成问题。

## 行政范围

Google 云平台管理范围通过 IAM 角色管理；然而，Google Cloud 身份默认的管理范围是基于域的，或者在[多域支持的情况下](https://support.google.com/a/answer/175747?hl=en&ref_topic=3541972)(忽略遗留的 G 套件品牌；这也适用于云身份)。您可以定义管理角色，这些角色属于一个[组织单位](https://support.google.com/a/answer/6129577?hl=en)(即。一组用户)以及[特定服务](https://support.google.com/a/answer/33325?hl=en&ref_topic=4514341)(例如，管理密码)(忽略传统的 G 套件品牌；这也适用于云身份)。

[超级管理员](https://support.google.com/a/answer/2405986)(忽略传统 G 套件品牌；这也适用于云身份)具有跨所有用户和云身份管理服务的完整范围(例如密码重置、[全域授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation))。这可能是按域或子域隔离客户，而不实施[多域支持](https://support.google.com/a/answer/175747?hl=en&ref_topic=3541972)的好理由。

## 缩放比例

每个客户的单个域或子域自然会带来可管理性的问题；这里有几个机制可以提供帮助:

*   您可以[将它们全部关联到一个账户](https://support.google.com/a/answer/7502379?visit_id=1-636560392711326124-2664298727&rd=1)，并通过[多域支持](https://support.google.com/a/answer/175747?hl=en&ref_topic=3541972)从同一个谷歌管理控制台管理它们(忽略传统的 G Suite 品牌；这也适用于云身份)。这简化了计费和管理，但也带来了一些限制:如果您正在实现 SAML 联合，那么域必须都共享同一个 SAML IDP 相同的管理员负责所有的领域，即。您不能按客户划分管理范围；由于所有域共享一个用户目录(尽管这可以通过子组织来划分)，用户隔离减少了。
*   [Admin SDK](https://developers.google.com/admin-sdk/) (忽略遗留的 G Suite 品牌；这也适用于云身份)提供了[创建新客户订阅](https://developers.google.com/admin-sdk/reseller/v1/reference/customers/insert)(例如[customerdomain.mydomain.com](http://chevron.slb.com/))的能力。使用此 API 需要您被启用为谷歌经销商。

## 演员表

您可以为每个域/组织分配至少一个 Google Cloud Platform 计费 id，或者跨域/组织共享计费 id。一如既往，每种方法都有利弊。

*   每个客户域/组织至少有一个 Google Cloud Platform 计费 id:这简化了传递计费，缓解了全域授权计费的不足。这也导致计费 id 激增。
*   跨域/组织共享的 Google 云平台计费 id:这最大限度地减少了计费 id 的增加，但实现起来很复杂，因为计费 id 属于最初创建它的组织，所以虽然您不能直接将其链接到多个组织，但您可以通过将其与创建项目的组织内的用户链接，将其用于不同组织内的项目。这种配置非常复杂，超出了本文的范围，并指出基于项目的隔离或多域支持(见上文)可能是更合适的选择。

# 基于网络的控制

## VPC 服务控制

正如我们在身份验证和授权文章中所讨论的，[域范围的授权](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)带来了数据泄漏的挑战。

[VPC 服务控制](https://cloud.google.com/vpc-service-controls/)(私有测试版)使您能够围绕云存储桶等谷歌云平台资源定义基于网络的安全边界，以将数据限制在虚拟私有云(VPC)内，并帮助降低数据泄露风险。

您的客户可以将资源放在安全区域，并建立基于 IP 的访问限制，这样他们的内部机密信息就不会与您的应用程序共享。

同样，如果选择项目作为客户隔离边界，可以使用 VPC 服务控件在多个租户项目之间建立数据隔离边界。您可以有选择地启用 VPC 区域之间的共享，例如在合资企业的情况下。

有一个关键的警告:VPC 服务控制是失效开放的，而不是失效关闭的，所以没有明确在 VPC 安全区的数据将可以跨越 VPC 的边界。强大的治理流程和自动化有助于降低这种风险。

# 下一步是什么

确定哪些选择最符合您的用例，并开始原型制作。

在 [API 浏览器](https://developers.google.com/apis-explorer/#p/)中试用 API。

阅读

*   [管理多个组织](https://cloud.google.com/resource-manager/docs/managing-multiple-orgs)学习如何将公司内的子组织或部门作为没有中央管理的独立实体来维护。
*   [谷歌云平台跨组织计费](/google-cloud/google-cloud-platform-cross-org-billing-41c5db8fefa6)了解如何跨多个 GCP 组织使用同一个计费账户。
*   [放心地共享数据:BigQuery 和 Data Studio 中的单元级访问控制](/google-cloud/share-data-with-confidence-cell-level-access-controls-in-bigquery-and-data-studio-cf753fa173a4)了解管理访问控制的方法，不仅在文档级别，而且针对单个结果。

# 承认

非常感谢:

*   Sam Srinivas 的身份表示思想，以及对多租户的广泛支持。
*   Samrat Ray 帮助我理解了 VPC 服务控制的细微差别。
*   罗伯·科奇曼帮我解决了租房的问题。