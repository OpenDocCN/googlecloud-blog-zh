# GCP 企业登录区|使用拒绝规则保持合规性的访问控制

> 原文：<https://medium.com/google-cloud/gcp-landing-zone-access-controls-using-deny-rules-to-maintain-compliance-16ea1e161522?source=collection_archive---------6----------------------->

身份和访问管理对于在任何组织中保持适当的安全态势都很敏感。过度宽松的角色可能会导致严重违规。攻击者只需利用 IAM 错误配置(如过于宽松的角色)就可以控制整个云环境。

作为任何云环境中的最佳实践，我们应该遵循“最低特权访问”。这说起来容易做起来难。

在企业登录区，我们大致有两个团队在登录区工作，即中央团队(基础架构团队/ SRE)和应用基础架构团队(SRE、DevOps)。

中央团队:中央团队负责维护网络、安全和核心基础设施，以便将任何应用基础设施装载到着陆区。中央团队将有特权访问，因为他们将负责核心基础设施。这些特权访问被给予一小组成员，且他们活动被监控。他们的主要目标是维护安全性、合规性和核心基础设施。

应用程序基础架构团队:根据着陆区内应用程序的数量，可以有多个应用程序基础架构团队。他们负责创建特定于其应用程序的基础架构。团队需要遵守中央团队设定的标准

下图显示了一个简单的着陆区设计，以证明两组之间的通道分布

![](img/bba7ba22fce06df5024acfa9589e663f.png)

图 1.0

在上图中，中央团队将负责共享的 VPC 主机项目。这将包括以下组成部分

1.  云互联
2.  云负载平衡器
3.  NGFW 虚拟机
4.  VPC、子网和 VPC 对等创建
5.  提供用户和子网对服务项目的访问

另一方面，应用程序基础结构团队将负责单独的服务项目。这包括:-

1.  在服务项目中创建 GKE、虚拟机、无服务器等服务

让我们假设上述架构是由一家公司部署的。根据公司的要求，应用程序基础架构团队不能在服务项目中创建任何网络组件。他们应该使用中央团队提供给他们的网络；此外，应用程序基础结构不应该能够向任何其他用户提供 IAM 访问。这应该是中央小组的责任

可以通过向各个团队授予最低权限访问来实现上述目的，以确保他们能够根据其职责进行访问，但请考虑以下情况

1.  应用程序团队中的一名用户被授予服务项目的计算管理员角色，以管理计算资源。这是中心组工程师的失误。他应该提供计算实例管理，但却提供了计算管理角色。计算管理员角色将使用户能够创建和更新服务项目中的 VPC 子网。因此，这将导致违反公司政策。
2.  中央团队被要求在开发环境中拥有最高的访问权限，以使团队能够更快地部署应用程序。中央团队为开发环境中的用户提供所有者访问权限。这导致了一个主要的问题，应用程序基础设施团队开始绕过中央团队向多个其他用户提供对开发环境的访问

GCP 最近启动了 IAM 的拒绝规则策略。([https://cloud.google.com/iam/docs/deny-overview](https://cloud.google.com/iam/docs/deny-overview)

我们可以利用拒绝规则来管理成熟的访问规则。使用拒绝规则策略，我们可以列出一组具有高度特权并且应该从应用程序基础结构团队中撤销的规则。

基于上述示例，这些可能包括:-

1.  计算网络*
2.  resource manager . projects . setiampolicy

这些规则可以应用于作为应用程序基础架构团队/非中央团队加入的所有用户。这将确保即使在 IAM 配置错误之后，也不会向他们提供特权访问。

重温上面讨论的场景，

现在，如果中心团队错误地向服务项目中的用户提供了计算管理角色。这一次，用户将无权创建或更新网络。将首先评估拒绝规则，并取消特权访问。因此，即使在 IAM 配置错误之后，也能保持系统完整

> *作者| Ankit Awal | Bhavish Kumar | Roshan Y*