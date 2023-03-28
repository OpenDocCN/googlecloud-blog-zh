# 给安全管理员的 12 份礼物

> 原文：<https://medium.com/google-cloud/12-gifts-for-the-security-admin-in-your-life-9c776aca1e7d?source=collection_archive---------1----------------------->

给自己做一杯温暖的饮料，在熊熊燃烧的炉火旁找一把相当于摇椅的东西，然后开始做一名与 GCP 一起工作的安全管理员应该在他们的节日礼盒中拥有的 12 样东西(由我选择)

1.[用例策略](https://cloud.google.com/solutions/policies/designing-gcp-policies) —本系列的每篇文章都使用一个假设的客户，并解释如何设计满足参考组织策略要求的 GCP 策略。他们帮助您解决问题，包括:

*   身份管理
*   组织映射:如何将您的组织结构映射到 GCP？
*   账单:你对账单有什么控制？你如何监控和理解支出？
*   网络配置:您的网络是否分隔并阻止需要分隔的区域之间的流量？
*   安全控制:如何以一种可以用 GCP 策略表达的方式实现安全控制

2.[云身份和访问管理](https://cloud.google.com/iam/) ( IAM)最佳实践——在这里，您可以找到资源来帮助您遵循[关于使用 GCP IAM 服务的最佳实践指南](https://cloud.google.com/iam/docs/using-iam-securely),以及如何实施 IAM 策略以映射到职能角色的示例([联网](https://cloud.google.com/iam/docs/job-functions/networking)和[计费](https://cloud.google.com/iam/docs/job-functions/billing)角色正在添加)使用示例场景来帮助您快速提升。

3.[组织策略服务](https://cloud.google.com/resource-manager/docs/organization-policy/overview) —该服务使您可以轻松实施适用于整个 GCP 组织的不断增长的策略列表。其中一个可用的策略允许您定义可以在资源上启用的一组服务及其 API。例如，您可以规定，除了数据工程师使用的项目之外，任何开发人员项目中都不允许使用 Spanner，因此您可以设置一个策略，不允许在放置所有非数据科学家项目的文件夹上使用 Spanner 服务。该文件夹中的所有项目都将继承该策略

*   可以按项目、文件夹或组织设置策略。
*   策略沿资源层次结构向下继承，如果组织的策略管理员允许，可以在设置组织策略的任何级别覆盖策略。

4.[云数据丢失防护](https://cloud.google.com/dlp/) —这项服务太棒了！这是一个单一的 API，可以帮助您[对您的数据进行分类，并编辑包含在文本文件中的敏感数据](https://cloud.google.com/dlp/docs/classification-redaction)流数据和存储在云存储和 BigQuery 等来源中的数据。[图像](https://cloud.google.com/dlp/docs/redacting-sensitive-data-images)对 DLP 的能力也没有抵抗力！

5.[云身份感知代理](https://cloud.google.com/iap/docs/concepts-overview) —作为一名安全管理员，您肯定会在某个时候担心到底是谁在访问您的 web 应用程序(可能一直都是！)Cloud IAP 负责认证和授权，因此只有通过认证的用户才有权访问应用程序。这是一项非常有趣的服务，是超越公司的敲门砖

6.防火墙——GCP 也有防火墙，但不像你所知道的那样。不同于传统的防火墙，在传统的防火墙中，你必须将流量引导到一个中间盒，而中间盒本身往往会成为一个阻塞点，GCP 在实例本身上执行防火墙规则。除了您习惯的标准来源和目的 IP 地址规则之外，您还可以根据服务帐户和标签来设置规则

7.[云审计日志](https://cloud.google.com/logging/docs/audit/) —帮助您回答“谁在何时何地做了什么？”

GCP 为您组织中的每个项目提供两个日志流

[管理活动日志](https://cloud.google.com/logging/docs/audit/#admin-activity)包含 API 调用或修改资源配置或元数据的其他管理操作的日志条目。管理活动日志始终处于启用状态。您的管理活动审计日志是免费的。

[数据访问日志](https://cloud.google.com/logging/docs/audit/#data-access)。数据访问审计日志，记录创建、修改或读取用户提供的数据的 API 调用。默认情况下，数据访问审计日志是禁用的，因为它们可能非常大。

这些日志与您的应用程序日志截然不同，但是 GCP 也为您提供了[堆栈驱动程序日志](https://cloud.google.com/logging/)和[堆栈驱动程序监控](https://cloud.google.com/monitoring/)，它们可以让您深入了解您的应用程序

8.[云 KMS](https://cloud.google.com/kms/) —是一项关键的管理服务。使用云 KMS，您可以在云托管的解决方案中管理对称加密密钥，无论它们是用于保护存储在 GCP 还是其他环境中的数据。你可以通过云 KMS API 创建、使用、旋转和销毁密钥，包括作为[秘密管理](https://cloud.google.com/kms/docs/secret-management)或[信封加密](https://cloud.google.com/kms/docs/envelope-encryption)解决方案的一部分。

9.管理 SSH 密钥——管理多个用户的 SSH 密钥并跟踪谁的密钥在哪里总是需要定义良好的流程。GCP 通过[操作系统登录 API](https://cloud.google.com/compute/docs/instances/adding-removing-ssh-keys#oslogin) 让这变得更加容易。这允许您将您的公共 SSH 密钥与来自 [G Suite](https://support.google.com/a/answer/4352075) 或 [Cloud Identity](https://support.google.com/a/answer/7319251) 的托管用户帐户相关联。它有以下好处:

*   公钥与用户帐户相关联，而不是与项目或实例元数据值相关联，后者更易于维护和跟踪。
*   可以通过单个 IAM 角色向用户授予 SSH 访问权限，而不必向这些用户授予元数据更新权限。
*   用户不能向元数据添加任意键，这使得审计谁有权访问实例变得更加容易。
*   可以通过撤销 IAM 角色来撤销单个帐户的 SSH 访问，而不是从元数据或实例本身中手动删除公共 SSH 密钥。

10.除了 IAM 特定的最佳实践指南，GCP 还提供了一些产品和服务特定的指导，以帮助您保护各种 GCP 资源。[管理桶和对象](https://cloud.google.com/storage/docs/best-practices#security)、[了解服务帐户](https://cloud.google.com/iam/docs/understanding-service-accounts)、[保护服务帐户密钥安全](https://cloudplatform.googleblog.com/2017/07/help-keep-your-Google-Cloud-service-account-keys-safe.html)(好吧，这是一篇博文！)是一些很好的开始。

11.作为一名安全管理员，检查您的云提供商遇到的控制通常意味着检查合规矩阵表和其他安全白皮书。

超级有用的矩阵(至少对我来说)是云安全合规矩阵，可以在这里找到和 [PCI DSS](https://cloud.google.com/security/compliance/pci-dss/) 共同责任[矩阵](https://cloud.google.com/files/PCI_DSS_Shared_Responsibility_GCP_v32.pdf)。除此之外，白皮书中还详细介绍了许多安全信息，可在[https://cloud.google.com/security/](https://cloud.google.com/security/)找到

12.最后但并非最不重要的[云功能](https://cloud.google.com/functions/) —这可能不是一个以安全为中心的产品，但作为一种对事件采取行动的方式是有用的。例如，您可能希望实现一个过程来防止上传到存储桶中的数据意外泄露，这些存储桶可能具有比文件分类所允许的更广泛的权限。通过将 DLP API 与云功能结合使用，您可以实施一个流程来隔离或删除 DLP 已扫描并归类为不符合您分配给存储桶的分类级别的文件。

节日问候