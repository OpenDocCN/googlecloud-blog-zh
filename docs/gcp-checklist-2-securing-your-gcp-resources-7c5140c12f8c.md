# GCP 清单 2 —保护您的 GCP 资源

> 原文：<https://medium.com/google-cloud/gcp-checklist-2-securing-your-gcp-resources-7c5140c12f8c?source=collection_archive---------1----------------------->

保护您的 GCP 环境需要解决您的用户被允许做什么；实施组织范围内的控制。通过实施适用的安全控制和保护部署在 GCP 的应用程序来保护您的 GCP 组织。

您需要考虑以下内容作为安全组织的基线:

*   管理身份—集中管理您的身份。利用组来简化管理。使用[操作系统登录](https://cloud.google.com/compute/docs/instances/managing-instance-access)来集中管理谁可以通过 SSH 进入您的虚拟机
*   InternationalAssociationofMachinists 国际机械师协会
*   使用最低特权原则。
*   如果预定义的 IAM 角色不适合您的使用情形，请创建自定义角色
*   如果使用自定义角色，首先从预定义的角色开始
*   请注意使用自定义角色的操作开销
*   为提升的角色(如组织管理员角色)实施碎玻璃访问，这将触发围绕这些角色的使用的更严格的审核。
*   使用组织策略实现组织范围的标准化
*   网络控制—允许您基于流量实施边界
*   子网根据子网范围提供逻辑边界。您必须明确允许子网之间的流量
*   管理源和目标之间允许的流量的防火墙
*   使用服务帐户在虚拟机之间配置防火墙规则
*   配置共享 VPC 以允许集中管理您的网络
*   使用安全区域提供纵深防御
*   保护基础设施——采用深度防御方法，利用 GCP 平台的功能，通过实施适用于您的使用案例的正确层的安全控制，向内工作
*   使用全局负载平衡和云装甲来保护您面向互联网的应用程序
*   使用 IAP 管理用户对面向 web 的应用程序的访问
*   使用 API 代理或者云端点或者 Apigee edge 来管理对 API 的认证调用
*   不要公开云存储桶使用 IAM 来管理访问
*   小心管理下载的安全密钥。实施轮换流程，避免机密意外加载到私有和公共回购中。
*   加密要求——如果允许 Google 管理您的加密需求不足以满足您的使用情况，那么您可以使用自己的密钥。使用 KMS 加密您的秘密和下载的密钥
*   保护数据—对数据进行分类—使用 DLP API 对数据进行分类和编辑，实施 iam 角色以限制对数据集的访问。审计数据访问。Dat 血统和位置都很重要。
*   库存——了解你的 GCP 资源。使用云安全指挥中心和/或 forseti
*   审计和警报—这些通常是出现问题的第一个迹象。使用 Stackdriver 来配置审计日志记录和设置警报
*   数据分类—使用 DLP API 对敏感数据进行分类和编辑
*   合规性—使用
*   事故响应(参见运营效率)
*   打破玻璃进入

这是你的阅读清单:

[DLP](https://cloud.google.com/dlp/docs/)

[我的角色](https://cloud.google.com/iam/docs/understanding-roles)

[BQ iam 角色](https://cloud.google.com/bigquery/docs/access-control)

[加密选择](https://cloud.google.com/security/encryption-at-rest/)

[安全站点](https://cloud.google.com/security/)

[合规现场](https://cloud.google.com/security/compliance/)

[IAM 更改日志](https://cloud.google.com/iam/docs/permissions-change-log)

[保护服务账户](https://cloudplatform.googleblog.com/2017/07/help-keep-your-Google-Cloud-service-account-keys-safe.html)

[检测安全密钥](https://cloud.google.com/source-repositories/docs/detecting-security-keys)

[网络控制](https://cloud.google.com/vpc/docs/)

[https://forsetisecurity . org/docs/latest/concepts/best-practices . html](https://forsetisecurity.org/docs/latest/concepts/best-practices.html)

当你完成后，这是你的清单:

该系列中的所有清单可在[这里](/@grapesfrog/using-gcp-theres-a-checklist-for-that-76d61d1ffcbc)找到