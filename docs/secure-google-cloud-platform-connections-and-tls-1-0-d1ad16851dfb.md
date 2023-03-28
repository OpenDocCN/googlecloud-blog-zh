# 保护谷歌云平台连接和 TLS 1.0

> 原文：<https://medium.com/google-cloud/secure-google-cloud-platform-connections-and-tls-1-0-d1ad16851dfb?source=collection_archive---------1----------------------->

禁用和缓解 TLS 1.0 连接

# 介绍

谷歌云平台(GCP)支持 TLS 1.0 以及 1.1 和 1.2；它将使用客户端浏览器支持的最高 TLS 级别执行身份验证握手。一些组织认为 TLS 1.0 不安全，并希望确保使用其他机制来保护连接。

本文提供了有关禁用和缓解谷歌云平台 TLS 1.0 认证的信息。

# 安全连接组件

本文档描述了满足这些要求的四个核心组件。

*   将浏览器配置为禁用 TLS 1.0。
*   使用专用网络来确保安全。
*   通过 Apigee 这样的代理公开公共 API。
*   代理被认为不安全的 Google 云平台服务，并将代理配置为禁用 TLS 1.0。

以下部分在基于 API 的管理访问和其他服务的安全特性的上下文中描述了这些组件。

# 基于 API 的管理访问

通过代理 API，如 Apigee 和 HTTPS/SSL 负载平衡器。

## **API gee API**

Apigee 是一个全生命周期的 API 管理平台，支持 API 提供商设计、保护、部署、监控和扩展 API。它用 OAuth 2.0、SAML、双向 TLS 保护传输中的数据，用加密保护静态数据； [TLS 1.0 和 1.1 已于 8 月](https://status.apigee.com/incidents/hk8ztlkpxynx)针对[北向流量](https://community.apigee.com/questions/57036/retiring-tls10-and-11.html)弃用。

## **云和开发者 API**

通过 [VPC 私有访问](https://cloud.google.com/vpc/docs/configure-private-google-access)用 [HTTPS/SSL 负载平衡器](https://cloud.google.com/load-balancing/)代理[这些 API](https://developers.google.com/apis-explorer/#p/)，并在负载平衡器上选择[受限配置文件。](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts#defining_an_ssl_policy)

# 其他服务的安全功能

## **客户端/浏览器**

使用现代浏览器并禁用 TLS 1.0。Chrome Enterprise 从 [Chrome 66](https://support.google.com/chrome/a/answer/7679408?hl=en) 开始支持 [SSLVersionMin 策略](https://www.chromium.org/administrators/policy-list-3#SSLVersionMin)。

请注意，基于客户端/浏览器的实施具有挑战性，尤其是在不同的浏览器环境中，例如对于远程和 SaaS 用户。

## **网络**

[谷歌云互联](https://cloud.google.com/interconnect/)提供你的网络和谷歌云平台之间的安全连接；您可以通过 VPN 等机制将这一点扩展到您的远程用户。这些机制可以在开放的公共网络上提供比 TLS 更安全的连接。

## **服务器**

**GCE 和** [**托管服务**](https://cloud.google.com/vpc/docs/private-access-options#pga-supported) **如 GCS**

在 HTTPS 负载平衡器上选择[“受限”配置文件以实施现代 TLS。](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts#defining_an_ssl_policy)

*   使用 [VPC 私有访问](https://cloud.google.com/vpc/docs/configure-private-google-access)，GCS 等托管服务可以与负载均衡器进行前端连接。

**GCE 上的 GKE/K8S**

GCE 上 GKE/K8S 的入口控制器利用 GCP HTTPS 负载平衡器；Ingress 目前不支持 SSL 配置文件的管理。

*   您可以在 HTTPS 负载均衡器上设置[“受限”配置文件，Ingress 支持这一点，但是这种行为不受支持，将来可能会改变，因此只能依靠适当的回归测试和治理。](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts#defining_an_ssl_policy)

Istio 使用 Envoy 负载平衡器。适当设置 [TLS 认可证书](https://www.envoyproxy.io/docs/envoy/latest/configuration/network_filters/client_ssl_auth_filter#config-network-filters-client-ssl-auth.)。

## 应用引擎标准和灵活

要么:

*   带有反向代理的前端应用引擎，如 NGINX 和[适当地配置 SSL 协议。](http://nginx.org/en/docs/http/configuring_https_servers.html)
*   为支持设置特定项目/域的应用引擎最低 TLS 级别建立案例(仅适用于自定义域，即不是*。[appspot.com](http://appspot.com/))。

**谷歌云端点**

*   云端点的 TLS 是由托管服务的平台提供的，因此请根据您的端点类型和代理，或者使用 Apigee 代理端点，参见上文。

# 下一步是什么

*   [PCI 数据安全标准合规架构指南](https://cloud.google.com/solutions/pci-dss-compliance-in-gcp)帮助您了解如何在谷歌云平台(GCP)上为您的企业实施支付卡行业数据安全标准(PCI DSS)。该指南提供了有关该标准的背景，解释了您在基于云的合规性中的角色，然后为您提供了使用 PCI DSS 设计、部署和配置支付处理应用程序的指南。本教程还讨论了监控、记录和验证应用程序的方法。
*   [谷歌云平台:PCI 客户责任矩阵](https://cloud.google.com/files/GCP_Client_Facing_Responsibility_Matrix_PCI_2018.pdf)可以作为您追求 PCI DSS 合规性和进行自己的 PCI DSS 审计的有用参考。