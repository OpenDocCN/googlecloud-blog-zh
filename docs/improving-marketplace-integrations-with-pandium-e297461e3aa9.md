# 改善与 Pandium 的市场整合

> 原文：<https://medium.com/google-cloud/improving-marketplace-integrations-with-pandium-e297461e3aa9?source=collection_archive---------2----------------------->

# 如何将我的软件推向所有市场？

集成各种各样的云服务、市场和产品很快会给你的开发团队带来大量的额外工作，使他们无法发布新功能。随着企业继续将他们的业务[向云](https://bluetree.ai/saas-statistics/)深处转移，大公司[平均使用 203 种 SaaS 产品。](https://www.blissfully.com/saas-trends/2019-annual/)您如何将所有这些不同的系统编织在一起？

> 接触 B2B 客户的地方太多了。

Pandium 帮助 SaaS 公司解决这个问题，让企业能够提供大规模的本地集成。这是一个专门设计的平台，旨在消除与构建和维护应用内集成市场相关的繁重工作。他们依靠谷歌云和谷歌 Kubernetes 引擎(GKE)来运行他们的平台，这些集成服务的易用性、可靠性和自动化使他们的工程团队能够减少对基础设施管理的关注，更多地关注核心产品的构建。

> 与多个第三方市场平台集成是一件麻烦的事情。

# **加快集成速度**

Pandium 的核心产品是为了应对本地集成市场日益增长的需求而产生的。*越来越少的业务用户希望等待他们自己的开发人员来构建定制集成*。越来越多的 SaaS 公司，如 [Salesforce](https://www.salesforce.com/solutions/appexchange/faq/) 、 [Shopify](https://apps.shopify.com/) 和 [Slack](https://pandiuminc.slack.com/apps) ，正在通过应用市场向他们的客户提供自助式的现成集成。

![](img/dbc16229294968dbb01c17bc7d86c73c.png)

图像[来源](https://pixabay.com/users/cocoparisienne-127419/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1798851)

从技术角度来看，公司在构建这些集成和大规模运行所需的基础设施方面面临挑战。他们的开发人员需要安全地构建、维护和托管这些集成，同时提供业务用户可以轻松导航的前端。这种构建极其复杂，因此，许多 SaaS 公司最终只能为客户提供很少的功能集成，维护和客户支持成本很高。

> 如果你的产品让开发者的生活更轻松，他们就会使用它。

Pandium 平台提供应用内市场基础设施——认证、安全、前端 UI、账户供应、商业用户登录和托管——因此 SaaS 开发者可以专注于编写他们客户需要的特定集成配置。

![](img/356997a7bf10fb5022aebf38d51f0ac3.png)

# **大多数平台集成都会产生开销**

传统的集成平台提供了一些无需代码即可实现的简单用例，但是对于更复杂的配置，它们需要开发人员在一个相当严格的系统中的可视化元素(捆绑代码)内部和周围进行编码。这意味着工程师必须学习一个只适用于自己的深奥系统。Pandium 旨在让开发人员通过他们的 repo 将简单的基于命令行的脚本(以他们已经编写的任何语言)安全地推送到 Pandium 平台，为工程师提供最大的灵活性和速度来迭代他们的集成配置，而不必学习新的系统。

> 与语言无关的命令行工具加速了您的工程设计。

为了使这项工作规模化，Pandium 的工程团队选择了一种带有容器的微服务架构，以确保它尽可能高效和安全地运行。有了这种结构，即使一个客户的集成出现问题，客户和他们的顾客也不会受到影响。由于 Pandium 在其平台上运行第三方代码，它面临着独特的安全问题，需要确保任何客户的错误或性能问题都不会影响任何其他客户的帐户。

![](img/413130b41acf046177593b3049721b49.png)

# **如果我的应用程序走红怎么办？**

此外，任何客户的应用内集成市场通常会迅速增加或减少用户，作为这些市场的主机，Pandium 需要一种能够有效应对快速变化的利用率而不影响可用性或导致成本大幅飙升的架构。

> 高效的自动扩展提高了信心和性能。

他们最初使用了不同的云提供商，他们花了几个小时来启动集群。他们还相当定期地收到与 kubernetes 控制平面相关的夜间页面，例如支持 kubernetes 控制平面的主 etcd3 数据库的高内存消耗。Pandium 决定*转到 GKE* ，这样他们可以专注于提供和管理大规模的集成市场，而把运行集群子系统的工作留给谷歌。

通过几个简单的 gcloud 命令，他们能够在几分钟内启动集群。此外，不必担心主节点的健康状况，这让他们能够在晚上睡得更好。

![](img/5ceea3dc3d38c87779142845b57045fd.png)

图片[来源](https://unsplash.com/@zvessels55?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

# **规模化运营，节约资金**

GKE 节点池可以设置为[可抢占](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms)，这样*在谷歌上运行一个集群可以节省 80%的成本*。在这种情况下，Pandium 以间断的时间间隔运行许多作业，它们不需要长时间运行。这意味着短期工作负载可以整合到他们的设置中，而不会影响客户的体验。

> 使用可抢占的 GKE 节点池节省 80%的费用。

此外，GKE 的节点池增加了客户需要的额外一层安全和隔离。例如，Pandium 的工作节点可以与客户端工作节点分开，并且与客户端节点相比，它们可以为它们完全控制的第一方节点运行不同的安全级别。

> 多租户系统需要强大的安全边界。所以，库伯内特斯！

从自动缩放和自动升级，到通过 Stackdriver 进行日志记录，GKE 使充分利用 Kubernetes 成为可能，而无需投入大量工程资源来设计、管理和维护其性能。

随着时间的推移，不同环境(即生产、开发和试运行)之间的差异变得太大，如果没有更具体的管理，就无法管理。集群部署时间从几分钟恢复到几小时。运营团队很难过。

![](img/ef4b205bc8adc9db2b4994e1fee80479.png)

图片[来源](https://unsplash.com/@goetz_heinen?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

# **加快部署= >让运营商更开心**

当寻求解决这个问题时，Terraform，特别是谷歌的预烘焙 Terraform 模块，允许 Pandium 用代码实现管理云基础设施。使用 Hashicorp 配置语言(HCL ),他们可以用清晰、简洁的代码来定义他们的基础设施，而不必弄清楚如何达到这种状态。这类似于针对数据库编写 SQL 查询。你声明你想要的结果，而不是如何得到回应的细节。这使得 Pandium 能够将他们的环境创建过程*缩短到几分钟*。

> Terraform 允许您用代码定义和部署基础设施。

有了谷歌提供的模块，他们比基础设施路线图提前了*个月，现在他们可以创建和配置一切，从云项目到 IAM 服务账户，再到 GKE 的节点和网络政策。以这种方式运行使 Pandium 即使在利用率高峰时也能确保高可用性，因为他们的系统可以自动伸缩。*

依靠谷歌的基础设施及其强大的支持，Pandium 已经能够为 SaaS 公司建立可扩展的应用内市场提供基础设施。这使得这些客户的开发人员可以进一步增强他们的核心产品，同时仍然为他们的客户提供他们需要的集成。要了解更多关于 Pandium 如何利用 GKE 的信息，请观看对 Pandium 首席执行官 Cristina Flaschen 的谷歌云采访。