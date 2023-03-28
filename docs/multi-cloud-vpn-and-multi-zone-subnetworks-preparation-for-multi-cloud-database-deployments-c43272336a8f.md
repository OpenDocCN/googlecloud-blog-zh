# 多云 VPN 和多区域子网—用于多云数据库部署的网络设置

> 原文：<https://medium.com/google-cloud/multi-cloud-vpn-and-multi-zone-subnetworks-preparation-for-multi-cloud-database-deployments-c43272336a8f?source=collection_archive---------0----------------------->

## 设置多云 VPN 的教程

# 介绍

这篇博客是关于如何在 Google 云和 AWS 之间建立 VPN，在每个云中创建多区域子网以及测试任意对任意连接的教程。对于设置的一部分，本博客遵循社区教程[AWS](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws)的 Google Cloud HA VPN 互操作性指南，并逐字引用。这个博客超越了社区教程的内容，包括设置子网以及对网络设置进行任意对任意的测试。

请预先注意，这是一个一步一步的手动设置，以防您想要理解并自己执行所有需要的步骤。如果你正在寻找一个自动设置，这个博客不适合你。例如，仅使用用户界面建立 VPN 的帖子是[这里是](/peek-travel/connecting-an-aws-and-gcp-vpc-using-an-ipsec-vpn-tunnel-with-bgp-f332c2885975)。

# 设置阶段

这篇博客将设置分为以下几个阶段:

*   **项目设置**。在 Google Cloud 中创建项目，在 AWS 中创建项目。这里不做进一步解释——这些说明假设您在每个云设置中都有一个项目，并且您有权根据需要设置 VPN、子网、防火墙规则、计算引擎和其他资源。
*   **VPN 配置**。初始设置阶段是在谷歌云和 AWS 之间设置一个 VPN。这是一套循序渐进的说明。
*   **子网配置**。下一阶段是设置子网以及防火墙和安全设置。
*   **测试**。最后一个阶段是通过在 Google Compute Engine 和 EC2 中创建虚拟机来测试设置，以进行任意对任意 ping 测试。

在成功的测试阶段之后，网络设置已经准备好，可以为多种云使用情形和工作负载进行网络设置。

# 注释

在谷歌云端显示了`glcoud`命令。您必须选择的变量值表示为`[…]`。它们在第一次使用时根据需要引入。您可以通过全局文本替换来一致地替换这些变量，以便一致地准备所有命令。

在 AWS 端，提供了用户界面说明，这些说明大部分是从[Google Cloud HA VPN inter operability guide for AWS](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws)中一字不差地摘录下来的，有时会做一些小的修改(为了澄清),并明确标记为`[[...]]`。

建议您记下所有选择的变量值以及在各种用户界面中输入的名称/值，以记录详细信息。下面是这种方法的一个例子:[配置参数和值](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws#configuration_parameters_and_values)。另一种方法是创建一个记录所有设置和变量值的电子表格。

# VPN 配置

## 在谷歌云项目中

*   创建一个网络(GCP:谷歌云平台):

```
gcloud compute networks create [GCP_NETWORK_NAME] \
--subnet-mode custom \
--bgp-routing-mode global
```

*   最佳做法是删除项目中的默认网络。这种删除不是必需的，但可以防止意外配置不正确的网络。如果云外壳指示以下情况，您可能必须先删除几个防火墙规则:

```
gcloud compute networks delete default// in case you need to delete firewall rules:
gcloud compute firewall-rules delete <firewall-rule-name>
```

*   添加两个子网:

```
gcloud compute networks subnets create [GCP_SUBNET_1] \
--network [GCP_NETWORK_NAME] \
--region [GCP_REGION_SUBNET_1] \
--range [GCP_RANGE_SUBNET_1]gcloud compute networks subnets create [GCP_SUBNET_2] \
--network [GCP_NETWORK_NAME] \
--region [GCP_REGION_SUBNET_2] \
--range [GCP_RANGE_SUBNET_2]
```

*   添加 VPN 网关:

```
gcloud compute vpn-gateways create [GCP_VPN_GATEWAY] \
--network [GCP_NETWORK_NAME] \
--region [GCP_REGION_SUBNET_1]
```

*   添加路由器:

```
gcloud compute routers create [GCP_ROUTER] \
--region [GCP_REGION_SUBNET_1] \
--network [GCP_NETWORK_NAME] \
--asn [GCP_ASN]
```

## 在 AWS 项目中

注:我在引用文本中的添加或修改用`[[...]]`表示，因此任何与[Google Cloud HA VPN operability guide for AWS](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws)的偏离都被明确标出。

*   创建具有默认租赁的 VPC
*   创建一个 AWS[[虚拟专用网]]网关，并将其连接到 VPC:

```
1\. In the AWS dashboard, under **Virtual Private Network**, select **Virtual Private Gateways**.2\. Click **Create Virtual Private Gateway**3.Enter a **Name** for the gateway.4\. Select **Custom ASN** and give the gateway an AWS ASN that doesn’t conflict with the [GCP_ASN].5\. Click **Create Virtual Private Gateway**. Click **Close**.6\. Attach the gateway to the AWS VPC by selecting the gateway you just created, clicking **Actions, Attach to VPC**.7\. Pull down the menu and select a VPC for this gateway and click **Yes, Attach**.8\. Note the ID of the gateway for the steps that follow.
```

*   创建站点到站点连接和客户网关:

```
1\. In the AWS dashboard, under **Virtual Private Network** select **Site-to-site VPN connections**.2\. Click **Create VPN connection**.3\. Enter a **Name** for the connection.4\. Pull down the **Virtual Private Gateway** menu and select the ID of the virtual private gateway you just created.5\. Under **Customer gateway** select **New**.6\. For **IP address**, use the public IP address that was automatically generated for Interface: 0 [[label of the interface in the GCP console]] of the HA VPN gateway you created earlier [[in GCP]].7\. For **BGP ASN**, use the value for [GCP_ASN].8\. Under **Routing options** make sure that **dynamic** is selected.9\. For **Tunnel 1** and **Tunnel 2**, specify the **Inside IP CIDR** for your AWS VPC network, which is a BGP IP address from a /30 CIDR in the 169.254.0.0/16 range. For example, 169.254.1.4/30\. This address must not overlap with the BGP IP address for the GCP side. When you specify the pre-shared key, you must use the same pre-shared key for the tunnel from the HA VPN gateway side. If you specify a key, AWS doesn’t autogenerate it.10\. Click **Create VPN connection**.
```

*   创建第二个站点到站点的连接(引用的文本还声明创建第二个 AWS 网关，但这是不可能的。所以我加了`[[not:...]]`表示不应该这样做):

```
Repeat the above steps to create a second [[not: AWS Gateway,]]  site-to-site connection, and customer gateway, but use the IP address generated for HA VPN gateway Interface: 1 [[(label of the interface in the GCP console)]] instead. Use the same [GCP_ASN].
```

*   下载每个站点到站点连接的 AWS 配置设置:

```
1\. As of this writing, you can download the configuration settings from your AWS virtual private gateway into a text file that you can reference when configuring HA VPN. You do this after you’ve configured the HA VPN gateway and Cloud Router.2\. On the AWS **VPC Dashboard** screen, in the left navigation bar, select **Site-to-Site VPN Connections**.3\. Check the checkbox for the first VPN connection to download.4\. At the top of the screen, click the middle button that reads **Download Configuration**. The **Download Configuration** button provides only one configuration file, the one for the selected connection.5\. In the pop-up dialog box, choose **vendor: generic**. There is no need to change the rest of the fields, Click the **Download** button.6\. Repeat the above steps, but choose the second VPN connection to download the file.
```

## 在谷歌云项目中

如下创建外部 VPN 网关资源(也称为对等 VPN 网关)。

当您为 AWS 虚拟专用网关创建 GCP 外部 VPN 网关资源时，您必须创建四个接口，如 AWS 拓扑图所示[[in here:[Google Cloud HA VPN inter operability guide for AWS](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws)]]。

注意:为了成功配置，您必须使用您先前下载的两个 AWS 配置文件中引用的 AWS 接口的公共 IP 地址。您还必须将每个 AWS 公共 IP 地址与特定的 HA VPN 接口完全匹配。下面的说明详细描述了这项任务。

使用以下命令创建外部 VPN 网关资源。按如下所述更换选项:

```
[[The instructions in 1\. ... 4\. are for the command following 4.:]]1\. For [AWS_GW_IP_0], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Outside IP address, Virtual private gateway**.2\. For [AWS_GW_IP_1], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Outside IP address, Virtual private gateway**.3\. For [AWS_GW_IP_2], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Outside IP address, Virtual private gateway**.4\. For [AWS_GW_IP_3], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Outside IP address, Virtual private gateway**.gcloud compute external-vpn-gateways create [GCP_PEER_GW_NAME] \
--interfaces \
0=[AWS_GW_IP_0],1=[AWS_GW_IP_1],2=[AWS_GW_IP_2],3=[AWS_GW_IP_3]
```

*   在 HA VPN 网关上创建 VPN 隧道

在之前创建的 HA VPN 网关上创建四个 VPN 隧道，每个接口两个。

在创建每个隧道的以下命令中，替换以下配置中注明的选项:

```
[[The instructions in 1\. ... 9\. are for the commands following 9.:]]1\. Replace [GCP_TUNNEL_NAME_IF0] with the name of the tunnel to [[e.g.]] tunnel-a-to-aws-connection-0-ip0.2\. Replace [GCP_TUNNEL_NAME_IF1] with the name of the tunnel to [[e.g.]] tunnel-a-to-aws-connection-0-ip1.3\. Replace [GCP_TUNNEL_NAME_IF2] with the name of the tunnel to [[e.g.]] tunnel-a-to-aws-connection-1-ip0.4\. Replace [GCP_TUNNEL_NAME_IF3] with the name of the tunnel to [[e.g.]] tunnel-a-to-aws-connection-1-ip1.5\. Replace [GCP_PEER_GW_NAME] with a name of the external peer gateway created earlier.6\. Replace [IKE_VERS] with 2\. Although the AWS virtual private gateway supports IKEv1 or IKEv2, using IKEv2 is recommended. All four tunnels created in this example use IKEv2.7\. Replace [AWS_SHARED_SECRET_0] through [AWS_SHARED_SECRET_3] with the shared secret, which must be the same as the shared secret used for the partner tunnel you create on your AWS virtual gateway. See [Generating a strong pre-shared key](https://cloud.google.com/network-connectivity/docs/vpn/how-to/generating-pre-shared-key) for recommendations. You can also find the shared secrets in the AWS configuration files that you downloaded earlier.8\. Replace [INT_NUM_0] with the number 0 for the first interface on the HA VPN gateway you created earlier.9\. Replace [INT_NUM_1] with the number 1 for the second interface on the HA VPN gateway you created earlier.
```

*   创建到 AWS 连接 0、IP 地址 0 的隧道:

```
gcloud compute vpn-tunnels create [GCP_TUNNEL_NAME_IF0] \
--peer-external-gateway [GCP_PEER_GW_NAME] \
--peer-external-gateway-interface 0 \
--region [GCP_REGION_SUBNET_1] \
--ike-version [IKE_VERS] \
--shared-secret [AWS_SHARED_SECRET_0] \
--router [GCP_ROUTER] \
--vpn-gateway [GCP_VPN_GATEWAY] \
--interface [INT_NUM_0]
```

*   创建到 AWS 连接 0、IP 地址 1 的隧道

```
gcloud compute vpn-tunnels create [GCP_TUNNEL_NAME_IF1] \
--peer-external-gateway [GCP_PEER_GW_NAME] \
--peer-external-gateway-interface 1 \
--region [GCP_REGION_SUBNET_1] \
--ike-version [IKE_VERS] \
--shared-secret [AWS_SHARED_SECRET_1] \
--router [GCP_ROUTER] \
--vpn-gateway [GCP_VPN_GATEWAY] \
--interface [INT_NUM_0]
```

*   创建到 AWS 连接 1、IP 地址 0 的隧道

```
gcloud compute vpn-tunnels create [GCP_TUNNEL_NAME_IF2] \
--peer-external-gateway [GCP_PEER_GW_NAME] \
--peer-external-gateway-interface 2 \
--region [GCP_REGION_SUBNET_1] \
--ike-version [IKE_VERS] \
--shared-secret [AWS_SHARED_SECRET_2] \
--router [GCP_ROUTER] \
--vpn-gateway [GCP_VPN_GATEWAY] \
--interface [INT_NUM_1]
```

*   创建到 AWS 连接 1、IP 地址 1 的隧道

```
gcloud compute vpn-tunnels create [GCP_TUNNEL_NAME_IF3] \
--peer-external-gateway [GCP_PEER_GW_NAME] \
--peer-external-gateway-interface 3 \
--region [GCP_REGION_SUBNET_1] \
--ike-version [IKE_VERS] \
--shared-secret [AWS_SHARED_SECRET_3] \
--router [GCP_ROUTER] \
--vpn-gateway [GCP_VPN_GATEWAY] \
--interface [INT_NUM_1]
```

*   分配 BGP IP 地址

按照以下说明将 BGP IP 地址分配给云路由器接口和 BGP 对等接口。

对于每个 VPN 隧道，从您之前下载的 AWS 配置文件中获取 AWS 和 GCP 的 BGP IP 地址和 ASN。

如下所示，替换 GCP 侧的选项:

```
[[The instructions in 1\. ... 4\. are for the commands following the next two blocks of instructions:]]1\. For [GCP_BGP_IP_0], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Customer gateway**.2\. For [GCP_BGP_IP_1], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Customer gateway**.3\. For [GCP_BGP_IP_2], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Customer gateway**.4\. For [GCP_BGP_IP_3], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Customer gateway**.
```

*   更换 AWS 侧的选项，如下所述:

```
1\. For [AWS_BGP_IP_0], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Virtual private gateway**.2\. For [AWS_BGP_IP_1], in the configuration file you downloaded for AWS Connection 0, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Virtual private gateway**.3\. For [AWS_BGP_IP_2], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #1, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Virtual private gateway**.4\. For [AWS_BGP_IP_3], in the configuration file you downloaded for AWS Connection 1, under **IPSec tunnel #2, #3 Tunnel Interface Configuration**, use the IP address under **Inside IP address, Virtual private gateway**.
```

*   [[For]] [AWS_PEER_ASN]在 BGP 第 4 小节下使用以下 ASN(目前 ASN 在所有四种情况下都是相同的):

```
1\. For [GCP_TUNNEL_NAME_IF0], in the AWS configuration file for AWS Connection 0, use the Virtual Private Gateway ASN under IPSec Tunnel #1.2\. For [GCP_TUNNEL_NAME_IF1], in the AWS configuration file for AWS Connection 0, use the Virtual Private Gateway ASN under IPSec Tunnel #2.3\. For [GCP_TUNNEL_NAME_IF2], in the AWS configuration file for AWS Connection 1, use the Virtual Private Gateway ASN under IPSec Tunnel #1.4\. For [GCP_TUNNEL_NAME_IF3], in the AWS configuration file for AWS Connection 1, use the Virtual Private Gateway ASN under IPSec Tunnel #2.gcloud compute routers add-interface [GCP_ROUTER] \
--interface-name [GCP_ROUTER_INTERFACE_NAME_0] \
--vpn-tunnel [GCP_TUNNEL_NAME_IF0] \
--ip-address [GCP_BGP_IP_0] \
--mask-length 30 \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-bgp-peer [GCP_ROUTER] \
--peer-name [GCP_BGP_PEER_NAME_0] \
--peer-asn [AWS_PEER_ASN] \
--interface [GCP_ROUTER_INTERFACE_NAME_0] \
--peer-ip-address [AWS_BGP_IP_0] \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-interface [GCP_ROUTER] \
--interface-name [GCP_ROUTER_INTERFACE_NAME_1] \
--vpn-tunnel [GCP_TUNNEL_NAME_IF1] \
--ip-address [GCP_BGP_IP_1] \
--mask-length 30 \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-bgp-peer [GCP_ROUTER] \
--peer-name [GCP_BGP_PEER_NAME_1] \
--peer-asn [AWS_PEER_ASN] \
--interface [GCP_ROUTER_INTERFACE_NAME_1] \
--peer-ip-address [AWS_BGP_IP_1] \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-interface [GCP_ROUTER] \
--interface-name [GCP_ROUTER_INTERFACE_NAME_2] \
--vpn-tunnel [GCP_TUNNEL_NAME_IF2] \
--ip-address [GCP_BGP_IP_2] \
--mask-length 30 \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-bgp-peer [GCP_ROUTER] \
--peer-name [GCP_BGP_PEER_NAME_2] \
--peer-asn [AWS_PEER_ASN] \
--interface [GCP_ROUTER_INTERFACE_NAME_2] \
--peer-ip-address [AWS_BGP_IP_2] \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-interface [GCP_ROUTER] \
--interface-name [GCP_ROUTER_INTERFACE_NAME_3] \
--vpn-tunnel [GCP_TUNNEL_NAME_IF3] \
--ip-address [GCP_BGP_IP_3] \
--mask-length 30 \
--region [GCP_REGION_SUBNET_1]gcloud compute routers add-bgp-peer [GCP_ROUTER] \
--peer-name [GCP_BGP_PEER_NAME_3] \
--peer-asn [AWS_PEER_ASN] \
--interface [GCP_ROUTER_INTERFACE_NAME_3] \
--peer-ip-address [AWS_BGP_IP_3] \
--region [GCP_REGION_SUBNET_1]
```

*   检查状态:

```
gcloud compute routers get-status [GCP_ROUTER] \
--region [GCP_REGION_SUBNET_1] \
--format='flattened(result.bgpPeerStatus[].name, result.bgpPeerStatus[].ipAddress, result.bgpPeerStatus[].peerIpAddress)'
```

在这一点上，我们不再遵循[Google Cloud HA VPN operability guide for AWS](https://cloud.google.com/community/tutorials/using-ha-vpn-with-aws)，因为该教程没有设置子网、虚拟机或多区域子网的防火墙规则以及跨这些子网的任何对任何 ping 测试。

## 在 AWS 项目中

*   在 AWS 控制台虚拟私有云下的**段路由表:**

```
1\. Select the route table for the VPC you created2\. If it is not named yet, give it a name for ease of identification3\. In the lower part, click the tab called **Route Propagation**4\. Select **Edit route propagation**5\. Check the checkbox under **Propagate**6\. Click **Save**
```

# 子网配置

## 在 AWS 项目中

在 AWS 中，子网是区域性资源，而在 Google Cloud 中，子网是区域性资源。对于 HA 部署，在 Google Cloud 中需要一个子网，因为该区域的所有区域都被覆盖。在 AWS 中，区域中的每个区域都需要为该区域中的资源建立自己的子网。

*   为 AWS 项目区域中的每个区域创建一个子网(如果您希望覆盖所有区域)。在**子网**部分:

```
1\. Click on **Create subnet**2\. Fill in the required information
```

为了测试连通性:

```
1\. Create VMs, one in each subnet. In order to ssh into an instance, it needs a public IP. 2\. When setting up VMs, create a new security group as it will have to be modified specifically for the VPN setup and it allows you to prevent open access from anywhere.3\. Modify the security group related to the EC2 instances in order to allow ingress SSH traffic from your laptop IP
```

另外还需要一个互联网网关:[https://docs . AWS . Amazon . com/VPC/latest/user guide/VPC _ 互联网 _ 网关. html](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) 。

*   创建一个互联网网关。在**部分，互联网网关**:

```
1\. Click **Create internet gateway**2\. Attached the internet gateway to the VPC. Select the internet gateway and click **Attach to VPC** in the **Actions** drop down menu3\. Select the VPC and attach it to the internet gateway
```

*   在**路由表**中:

```
1\. Select the route table of the VPC2\. In the tab **Subnet Assocations** associate all subnetworks to the routing table of the VPC3\. In the tab **Routes** add the route from 0.0.0.0/0 to the internet gateway
```

为了允许 AWS EC2 实例相互 ping 通，AWS 安全组需要入口规则，以便允许来自所有子网的入口。为子网添加入口规则后，您可以使用每个实例的私有 IP 地址从每个实例 ping 每个实例。对于 ping，ICMP 入口就足够了。

为了允许 GCP VM 实例 ping AWS EC2 实例，在 EC2 实例的 AWS 安全组上，要添加两个入口规则，每个 Google Cloud 子网一个。

安全组中总共应该有几个进入规则:

*   每个 AWS 子网一个
*   每个 GCP 子网一个
*   一个供您的笔记本电脑访问 AWS EC2 虚拟机

此时，您可以 ssh 到您创建的 EC2 实例，并从另一个实例 ping 另一个实例。

## 在谷歌云项目中

上面创建了两个不同区域中的两个子网。这种设置允许两个区域中的资源相互交互，以便例如实施灾难恢复策略。

*   为了允许资源通信，必须实施以下防火墙规则:

```
gcloud compute firewall-rules create default-allow-ssh \
--network [GCP_NETWORK_NAME] \
--allow tcp:22 \
--source-ranges 0.0.0.0/0gcloud compute firewall-rules create default-allow-icmp \
--network [GCP_NETWORK_NAME] \
--allow icmp \
--source-ranges 0.0.0.0/0
```

一种立即测试连通性的方法是在两个区域中的每个区域创建一个虚拟机，并让它们互相 ping 通。如果有效，防火墙规则设置正确。

# 测试

此时，您已经理想地配置了:

*   每个 GCP 子网中有一台虚拟机
*   每个 AWS 子网中有一个虚拟机

从测试的角度来看，在 Google Cloud 和 AWS 中，每个虚拟机都应该能够 ping 通自己和其他虚拟机(也就是完整的网格可达性)。

# 其他连接选项

VPN 不是唯一的连接选项。根据需求，尤其是带宽和性能，存在 VPN 的替代方案。参见这篇博客[连接到谷歌云:你的网络选项解释](https://cloud.google.com/blog/products/networking/google-cloud-network-connectivity-options-explained)以及本页[谷歌云混合连接](https://cloud.google.com/hybrid-connectivity) y 了解详情和决策树。

# 摘要

上面的说明

*   在 Google 云和 AWS 之间设置 VPN
*   在谷歌云中的两个区域各创建了一个子网
*   根据您的选择在不同的 AWS 区域创建了一个或多个子网
*   测试了 Google 云和 AWS 中虚拟机的连接性。

此时，您已经有了一个多云、多区域和多分区的设置。您可以在子网中部署额外的资源。如果在资源之间需要额外的网络协议，不要忘记相应地设置防火墙规则。

# 承认

我要感谢阿尼巴尔·圣地亚哥([阿尼巴尔·圣地亚哥](https://medium.com/u/e995c6698d2d?source=post_page-----c43272336a8f--------------------------------))为提高这一内容的准确性所做的全面审查和许多评论。

# 放弃

Christoph Bussler 是谷歌公司(Google Cloud)的解决方案架构师。这里陈述的观点是我自己的，而不是谷歌公司的。