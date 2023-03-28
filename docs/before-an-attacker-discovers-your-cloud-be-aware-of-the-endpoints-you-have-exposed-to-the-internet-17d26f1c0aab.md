# 在攻击者发现您的云之前，请注意您已经暴露在互联网上的端点。

> 原文：<https://medium.com/google-cloud/before-an-attacker-discovers-your-cloud-be-aware-of-the-endpoints-you-have-exposed-to-the-internet-17d26f1c0aab?source=collection_archive---------3----------------------->

![](img/285b3c15de40be354a82a7e9a3f10282.png)

攻击者不断扫描互联网端点，寻找容易被访问和破坏的服务。一旦一个公共 IP 变得活跃，请记住，攻击者和不同的搜索引擎开始抓取它。在妥协变得不可避免之前，跟踪您的云中哪些端点可以通过互联网访问并定期审核这些端点始终是一个好主意。

GCP 上的 VPC 服务为各种服务(如 VM/Kube 服务、CloudSQL、负载平衡器等)提供网络功能，这些服务可用于通过附加外部 IP 地址在互联网上公开来运行服务。

GCP 帮助跟踪/查看项目中使用的所有外部地址列表，使其与各自的资源保持一致。让我们看看我们如何利用可用的服务和 API 来查找服务，即在相同的上向互联网公开的服务。

用导航菜单下的“网络”->“VPC 网络”并选择“IP 地址”

![](img/ec66524ed9edc8d41bbe7c6200ec81ca.png)

我们不会关注没有被任何资源利用的保留静态地址；相反，我们将简单地浏览项目中使用的可通过互联网访问的“外部 IP 地址”列表。我们将检查实例和任何其他组件使用的服务，如负载平衡器、Kube 或其他服务，而不考虑“静态/短暂”类型。
要访问 IP 地址服务，拥有“roles/compute.networkViewer”权限就足够了。

![](img/8c3287cb1ed8428f875b4786cc141563.png)

上图中，重点关注服务使用的地址非常重要。“使用中”中的短语“无”指的是保留的、未被任何组件使用的地址，但您仍然需要为它们付费。
要编译给定平台上所有项目或特定项目的地址列表，可以利用 compute [REST API](https://cloud.google.com/compute/docs/reference/rest/v1/addresses) 。

您可以使用以下 github 资源库中的 python 脚本来使事情变得更加简单；关于用法的更多信息

```
git clone https://github.com/smmadhu/gcp_external_ip_audit.git
```

克隆回购，并使用 google 应用默认登录“g cloud auth application-default log in”[引用](https://cloud.google.com/docs/authentication/provide-credentials-adc#local-dev)或服务帐户密钥在本地执行。您可以使用它来获取组织内特定项目或所有项目的地址。

我们已经得到了项目中使用的所有外部地址，接下来呢？
一旦我们有了 GCP 上使用的所有外部 IP 地址列表，就扫描互联网上暴露的端口，并在暴露的端口和端点上执行原因审计。可能会有这样的情况，ssh/sql 端口之类的敏感服务可能会被某个人无意中打开。了解暴露了哪些端口以及它们背后运行了哪些服务版本来缓解服务上的已知漏洞是很好的。

*   对项目中使用的外部 IP 地址进行定期审核。
*   扫描所有外部地址和服务上打开的端口，以检查是否有任何漏洞正在使用的服务版本。
    注意:对于全局负载平衡器地址，扫描时会有多个开放端口，更多详情可在[这里](https://cloud.google.com/load-balancing/docs/https#open_ports)找到。
*   作为软件开发过程的一部分，对公开的服务执行测试，以查找应用程序级别的任何安全漏洞。
*   避免限制服务在互联网上的公开，如果他们不是真的打算这样做的话。

注意:通过上面收集的数据，将会有由 google cloud 本身管理的地址，如路由器(用于云路由器)和包含 esp/udp500/4500(用于 VPN 网关地址)的转发规则。扫描时可以忽略这些条目。

通过定期监控和扫描我们的云环境中暴露的端点，让我们确保我们不会成为攻击的目标。

希望这有帮助…