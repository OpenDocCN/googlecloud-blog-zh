# 保护和审计谷歌云平台边界

> 原文：<https://medium.com/google-cloud/secure-and-audit-the-google-cloud-platform-perimeter-a3c33f451a82?source=collection_archive---------4----------------------->

## 审计

这篇文章描述了谷歌云平台如何解决在[概念文章](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-685466c62d0f)中描述的以下传统外围安全问题:如何审计流量和数据访问，即如何知道控制是否按预期工作？

Google Cloud Platform 提供了许多审计服务，这些服务与之前文章中描述的解决方案组件相对应…

# 负载平衡日志(alpha)

[HTTP(S)负载平衡日志](https://cloud.google.com/load-balancing/docs/https/https-logging-monitoring)包含大多数 GCP 日志以及 HttpRequest 日志字段中显示的一般信息。

局限性:

*   该产品处于 alpha 阶段。
*   未填充 HttpRequest.protocol。

# 应用引擎 HTTP 请求日志

[App Engine HTTP 请求日志](https://cloud.google.com/appengine/articles/logging)记录发送到所有 App Engine 标准和灵活应用的请求，默认提供。您可以在 App Engine 灵活环境中使用应用程序日志来补充这些内容。

如果使用反向代理，如 NGINX，为最终用户 IP 添加一个 HTTP 头，以便能够在应用引擎请求日志中显示它。

# VPC 流量测井

[VPC 流日志](https://cloud.google.com/vpc/docs/using-flow-logs)记录 VM 实例发送和接收的 TCP 和 UDP 网络流的样本。这包括 RDP 流量，因为它是 TCP(有时是 UDP)。

限制:

*   VPC 流日志仅位于虚拟机的下游。
*   它们对托管数据服务(如谷歌云存储)访问提供了有限的洞察力。

# 下一步是什么

阅读以下内容，了解本文中描述的概念和解决方案组件的更多信息:

*   [企业组织的最佳实践:日志记录、监控和审计](https://cloud.google.com/docs/enterprise/best-practices-for-enterprise-organizations#logging-monitoring-auditing)。
*   [HTTP(S)负载平衡日志](https://cloud.google.com/load-balancing/docs/https/https-logging-monitoring)
*   [应用引擎 HTTP 请求日志](https://cloud.google.com/appengine/articles/logging)。
*   [VPC 流量日志](https://cloud.google.com/vpc/docs/using-flow-logs)。

阅读以下内容以了解:

*   [谷歌云平台容器和虚拟机威胁检测与防护](https://medium.com/p/a40ef4403bca/edit)