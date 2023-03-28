# 保护和审计谷歌云平台边界

> 原文：<https://medium.com/google-cloud/secure-and-audit-the-google-cloud-platform-perimeter-ade393c25467?source=collection_archive---------3----------------------->

## 受限 IP 范围

这篇文章描述了谷歌云平台如何解决在[概念文章](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-685466c62d0f)中描述的以下传统外围安全问题:如何确保您的用户只与您或您的 SaaS 提供商的应用程序通信？

企业通常通过仅支持与白名单 IP 的通信来保护自己的边界；这要求 IP 是已知的，并且是有限的。谷歌云平台提供了许多支持这一点的服务…

# 发布的 IP 范围

谷歌云[公布其 IP 范围](https://cloud.google.com/compute/docs/faq#where_can_i_find_product_name_short_ip_ranges)。

限制

*   这包括所有的谷歌云服务，所以你不能区分 G Suite 和 App Engine。
*   您必须动态更新受限 IP 范围，因为它们会随着时间的推移而变化。

# 通过谷歌云负载平衡的单个全球或区域 IP

[谷歌云负载平衡](https://cloud.google.com/load-balancing/docs/load-balancing-overview)支持单个全球或地区 IP。

局限性:

*   Google 云存储仅支持 XML API。
*   非私有访问反向代理通过公共 IP 路由到应用引擎，但不通过公共互联网。

# 通过反向代理如 NGINX 的单一 IP

这在功能和限制上类似于 Google 云负载平衡。

*   *私有 K8S 工作节点:[设置私有集群](https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters)
*   ** IP 白名单:[为主访问配置授权网络](https://cloud.google.com/kubernetes-engine/docs/how-to/authorized-networks)
*   *** [VPC 私人访问](https://cloud.google.com/vpc/docs/configure-private-google-access)

下表描述了这些解决方案组件如何跨概念文章中描述的代表性 Google 云平台服务支持受限 IP。

*   * [计算引擎 IP 范围](https://cloud.google.com/compute/docs/faq#where_can_i_find_product_name_short_ip_ranges)
*   ** [应用引擎 IP 范围](https://cloud.google.com/appengine/kb/)
*   * * *[NGINX 控制器入口教程](https://cloud.google.com/community/tutorials/nginx-ingress-gke)

# B2B SaaS 皱纹

作为一家 SaaS 提供商，您可能已经完全接受并利用了谷歌云平台的深度防御功能，并且不愿意承担增加效用有限的外围防御层的管理负担，特别是当主要目标是保护客户的网络免受互联网攻击，而不是保护您的 SaaS 应用程序时，例如受限 IP。

在这种情况下，一种选择是让那些需要这种能力的客户在他们自己的谷歌云平台项目中实现谷歌云负载平衡或反向代理，并在云中路由到您的 SaaS 应用程序。

# 下一步是什么

阅读以下内容，了解本文中描述的解决方案组件的更多信息:

*   [谷歌云负载均衡](https://cloud.google.com/load-balancing/docs/load-balancing-overview)。
*   [VPC 私人访问](https://cloud.google.com/vpc/docs/configure-private-google-access)。

阅读以下指南，了解谷歌云平台在以下周边安全领域的能力。

*   [审计](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-a3c33f451a82)。