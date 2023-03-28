# 保护和审计谷歌云平台边界

> 原文：<https://medium.com/google-cloud/secure-and-audit-the-google-cloud-platform-perimeter-d16372fa6697?source=collection_archive---------2----------------------->

## “私人”运输

这篇文章描述了谷歌云平台如何解决在[概念文章](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-685466c62d0f)中描述的以下传统边界安全问题:如何保护用户向云的过渡？

谷歌云平台提供了许多支持这一点的服务，除了默认的 HTTPS…

## **云 VPN**

[云 VPN](https://cloud.google.com/vpn/docs/concepts/overview) 通过 IPsec VPN 连接将您的内部网络安全地连接到您的谷歌云平台(GCP)虚拟专用云(VPC)网络。在两个网络之间传输的流量由一个 VPN 网关加密，然后由另一个 VPN 网关解密。

局限性:

*   管理规模
*   不支持谷歌云负载均衡器，也不直接支持 App Engine、谷歌云存储等托管服务。

## **直接和运营商对等**

[直接](https://cloud.google.com/interconnect/docs/how-to/direct-peering)和[运营商](https://cloud.google.com/interconnect/docs/how-to/carrier-peering)对等通过我们的一个覆盖广泛的 Edge 网络位置提供到 Google 和 Google Cloud properties 的专用链接。

局限性:

*   没有 SLA

## **专用和合作伙伴互连**

[专用](https://cloud.google.com/interconnect/docs/concepts/dedicated-overview)和[合作伙伴互连](https://cloud.google.com/interconnect/docs/concepts/partner-overview)在您的内部网络和 Google 网络之间提供直接的物理连接。

局限性:

*   不支持[谷歌云负载平衡](https://cloud.google.com/load-balancing/)或者直接支持 App Engine、谷歌云存储等托管服务。
*   专用互连最小带宽要求。

## 来自反向代理的私有 IP 访问

[云负载平衡](https://cloud.google.com/load-balancing/)和反向代理如 [NGINX](https://www.nginx.com) 可以把你的资源放在单个任播 IP 后面，私下连接到谷歌云平台后端。

局限性:

*   反向代理通过公共 IP 路由到应用引擎，但不通过公共互联网。
*   Kubernetes / Google Kubernetes 引擎主 IP 白名单是测试版。

下表描述了这些解决方案组件如何支持概念文章中描述的代表性 Google 云平台服务之间的“私有”传输。

*   *私有 K8S 工作节点:[设置私有集群](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters)
*   ** IP 白名单:[为主访问配置授权网络](https://cloud.google.com/kubernetes-engine/docs/how-to/authorized-networks)
*   *** [VPC 私人访问](https://cloud.google.com/vpc/docs/configure-private-google-access)

# 下一步是什么

阅读以下内容，了解本文中描述的概念和解决方案组件的更多信息:

*   企业组织的最佳实践:网络和安全。
*   [谷歌云负载均衡](https://cloud.google.com/load-balancing/)。
*   [NGINX](https://www.nginx.com) 。
*   [谷歌云 VPN](https://cloud.google.com/vpn/docs/concepts/overview) 。
*   [谷歌云互联](https://cloud.google.com/interconnect/)。
*   [VPC 私人访问](https://cloud.google.com/vpc/docs/configure-private-google-access)。

阅读以下指南，了解谷歌云平台在以下周边安全领域的能力。

*   [受限 IP 范围](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-ade393c25467)。
*   [审计](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-a3c33f451a82)。