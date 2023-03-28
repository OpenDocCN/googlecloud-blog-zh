# 谷歌云平台容器和虚拟机威胁检测和保护

> 原文：<https://medium.com/google-cloud/google-cloud-platform-container-threat-detection-and-protection-a40ef4403bca?source=collection_archive---------1----------------------->

## 安全扩展

迁移到谷歌云平台的优势之一是安全性被编织到云的结构中。尽管如此，大多数大型企业已经围绕其内部工作负载开发了强大的威胁检测和保护生态系统，并希望将一些明确的洞察力和控制力带到云中。

[谷歌云的目标之一](https://cloud.google.com/why-google-cloud/)是在你所在的地方与你见面，而[谷歌安全设计的核心构件之一](http://services.google.com/fh/files/misc/csuite_security_ebook.pdf)是…

> Google Cloud Platform 的基础设施安全性是按递进层次设计的——硬件、服务、用户身份、存储、互联网通信和运营。我们称之为深度防御。

上面的文章描述了这些层，包括谷歌定制设计的芯片[泰坦硬件安全芯片](https://cloud.google.com/blog/products/gcp/titan-in-depth-security-in-plaintext)，允许谷歌在硬件层面识别和认证合法的谷歌设备。

以下内容探讨了一些可用于检测和防范跨越[谷歌计算引擎](https://cloud.google.com/compute/)实例、 [Kubernetes 容器](https://kubernetes.io/docs/concepts/containers/)和[谷歌应用引擎](https://cloud.google.com/appengine/) [](https://cloud.google.com/compute/)的威胁的控件。

# KVM 虚拟机管理程序

Google Cloud 使用开源的 KVM 虚拟机管理程序作为 Google Compute Engine 和 Google Container Engine 的基础，并根据其研究和测试经验投资于额外的安全强化和保护。然后，它反馈 KVM 项目的变更，使整个开源社区受益。

以下是 Google security 加固 KVM 以帮助提高应用程序安全性的主要方式:

*   主动漏洞搜索
*   减少攻击表面积
*   非 QEMU 实现
*   引导和作业通信
*   代码出处
*   快速而优雅的漏洞响应
*   小心控制的释放

点击此处了解每一项措施背后的更多安全措施[。](https://cloud.google.com/blog/products/gcp/7-ways-we-harden-our-kvm-hypervisor-at-google-cloud-security-in-plaintext)

# 计算引擎

## GCE 屏蔽虚拟机(测试版)

[屏蔽虚拟机](https://cloud.google.com/shielded-vm/)是谷歌云平台上的虚拟机，通过一系列安全控制措施进行强化，有助于抵御 rootkits 和 bootkits。

使用屏蔽虚拟机有助于保护企业工作负载免受远程攻击、权限提升和恶意内部人员等威胁。受保护的虚拟机利用高级平台安全功能，如安全和测量启动、虚拟可信平台模块(vTPM)、UEFI 固件和完整性监控。

## GCE 信任的图像

[GCE 可信映像 IAM 策略](https://cloud.google.com/compute/docs/images/restricting-image-access)允许您限制您的项目成员，使他们只能从包含符合您的策略或安全要求的认可软件的映像创建启动盘。您可以定义一个组织策略，允许您的项目成员仅从特定项目中的映像创建永久磁盘。

# Docker、Kubernetes 和 Google Kubernetes 引擎

## 集装箱安全概述

[集装箱安全概述](https://cloud.google.com/containers/security/)描述了如何在三个关键领域保护您在 GCP 的集装箱环境:

*   基础设施安全
*   软件供应链
*   运行时安全性

## 基础设施安全

**探索容器安全性:用 Kubernetes 引擎 1.10 运行一艘严密的船**

[本文](https://cloud.google.com/blog/products/gcp/exploring-container-security-running-a-tight-ship-with-kubernetes-engine-1-10)提供了强化 Kubernetes 引擎集群的最佳实践，并更新了 Kubernetes 引擎版本 1.9 和 1.10 中的新特性。

**容器优化操作系统**

[Google 的容器优化操作系统](https://cloud.google.com/container-optimized-os/docs/concepts/features-and-benefits)是一个针对计算引擎虚拟机的操作系统映像，针对安全运行 Docker 容器进行了优化。它是 Google 云平台上 [Kubernetes 引擎](https://cloud.google.com/kubernetes-engine/)和其他 [Kubernetes](https://kubernetes.io/) 部署中的默认节点 OS 镜像。您还可以使用容器优化操作系统，通过最少的设置，在计算引擎实例上快速启动 Docker 容器。

它提供了以下安全优势:

*   更小的攻击面:容器优化的操作系统占用空间更小，减少了实例的潜在攻击面。
*   默认锁定:默认情况下，容器优化的操作系统实例包括锁定的防火墙和其他安全设置。它阻止安装第三方内核模块或驱动程序。
*   自动更新:容器优化的 OS 实例被配置为在后台自动下载每周更新；要使用最新的更新，只需重新启动。

## 软件供应链

**帮助保护谷歌 Kubernetes 引擎上的软件供应链**

[这篇文章](https://cloud.google.com/solutions/secure-software-supply-chains-on-google-kubernetes-engine)向你展示了在你的代码被部署到谷歌 Kubernetes 引擎(GKE)集群之前，如何确保你的软件供应链遵循一条已知的安全路径。本文回顾了二进制授权是如何工作的，然后解释了如何通过谷歌云平台(GCP)最好地实现和使用它，以确保您的部署管道可以提供尽可能多的信息，帮助您在每个所需的阶段执行批准。

**谷歌容器注册表图片分析(测试版)**

[容器分析](https://cloud.google.com/container-registry/docs/get-image-vulnerabilities) [为容器注册表中的 Ubuntu、Debian 和 Alpine 映像提供](https://cloud.google.com/container-registry/docs/container-analysis#vulnerability_source)包漏洞扫描，并根据外部 [CVE 数据](https://cve.mitre.org/)来源分配[通用漏洞评分系统(CVSS)](https://www.first.org/cvss/) 分数。

容器分析支持初始、增量和连续扫描。

## 运行时安全性

**gVisor 容器沙盒(开源)**

[gVisor](https://cloud.google.com/blog/products/gcp/open-sourcing-gvisor-a-sandboxed-container-runtime) 为沙盒化不受信任的 Docker 和 Kubernetes 工作负载提供了快速且经济高效的解决方案，使在生产环境中运行沙盒容器变得简单易行。沙箱将防止潜在的危害在容器之间传播；用沙箱保护不受信任的应用程序和程序是保持系统安全可靠的最佳实践之一。Google 建议使用独立的集群和节点来隔离 Kubernetes 引擎上的可信和不可信工作负载。

因为它在更深的级别上沙箱化不可信工作负载，所以您可以在同一节点上部署可信工作负载和沙箱化不可信工作负载。这将简化您的工作负载管理，并允许您优化利用集群中的资源:您可以在一个节点、一个集群和一个项目中拥有多个沙盒，而无需花费很长的启动时间来创建集群和虚拟机，也无需在单独的集群和节点中利用 Kubernetes 集群中固有的未充分利用的资源。

**二进制授权(测试版)**

[二进制授权](https://cloud.google.com/binary-authorization/)是一种部署时安全控制，确保只有可信的容器映像被部署在 Kubernetes 引擎上。使用二进制授权，您可以要求在开发过程中由可信机构对图像进行签名，然后在部署时实施签名验证。通过实施验证，您可以确保部署到特定生产环境的所有容器映像都符合部署策略，从而获得对容器环境的更严格控制。例如，您可以要求部署到 prod-payment 集群的所有映像由您的集中构建者、QA 团队和试运行测试进行签名。

# 云安全扫描器

[云安全扫描器](https://cloud.google.com/security-scanner/)是针对 Google App Engine Standard、[以及](https://cloud.google.com/security-scanner/docs/scanning-compute-engine)(在 alpha 中)Google Compute Engine 和 Kubernetes Engine 中常见漏洞的 web 安全扫描器。它可以自动扫描和检测常见漏洞，包括跨站点脚本(XSS)、Flash 注入、混合内容(HTTPS 的 HTTP)、明文密码和过时/不安全的 Javascript 库。

云安全扫描器使您能够在生产之前检测开发中的关键漏洞；设置扫描后，它会自动搜索应用程序，跟踪起始 URL 范围内的所有链接，并尝试执行尽可能多的用户输入和事件处理程序。您可以选择是否使用 Chrome、Safari、Blackberry 或诺基亚浏览器代理。

# 云安全指挥中心(测试版)

[云安全指挥中心](https://cloud.google.com/security-command-center/)帮助安全团队收集数据，识别威胁，并在威胁导致业务损害或损失之前采取行动。它提供对应用程序和数据风险的深入洞察，以便您可以快速缓解对云资源的威胁并评估整体运行状况。

它通过谷歌开发的内置异常检测技术，帮助识别僵尸网络、加密货币挖掘、异常重启和可疑网络流量等威胁。

它还整合了云安全扫描器等服务的安全见解，以及来自供应商的第三方云安全解决方案，如 [Twistlock](https://www.twistlock.com/) 和 [Redlock](https://redlock.io/platform/technology) 。

云安全指挥中心为[发布/订阅集成提供了配套应用，可用于触发云功能](https://cloud.google.com/security-command-center/docs/how-to-pubsub-and-cloud-functions)进行补救。

# 下一步是什么

阅读以下内容以了解实施情况:

*   [探索容器安全性:用 Kubernetes 引擎 1.10 运行一艘严密的船](https://cloud.google.com/blog/products/gcp/exploring-container-security-running-a-tight-ship-with-kubernetes-engine-1-10)
*   [帮助保护谷歌 Kubernetes 引擎上的软件供应链](https://cloud.google.com/solutions/secure-software-supply-chains-on-google-kubernetes-engine)

阅读以下内容，了解本文中描述的概念和解决方案组件的更多信息:

*   [纵深防御](https://paidpost.nytimes.com/google-cloud/defense-in-depth.html)
*   [安全电子书的谷歌云方案](http://services.google.com/fh/files/misc/csuite_security_ebook.pdf)
*   [深度泰坦:明文安全](https://cloud.google.com/blog/products/gcp/titan-in-depth-security-in-plaintext)
*   [我们在 Google Cloud 强化 KVM 虚拟机管理程序的 7 种方式:明文安全](https://cloud.google.com/blog/products/gcp/7-ways-we-harden-our-kvm-hypervisor-at-google-cloud-security-in-plaintext)
*   [屏蔽的虚拟机](https://cloud.google.com/shielded-vm/)
*   [GCE 可信图像](https://cloud.google.com/compute/docs/images/restricting-image-access)
*   [容器优化操作系统](https://cloud.google.com/container-optimized-os/docs/concepts/features-and-benefits)
*   [谷歌容器注册表图片分析](https://cloud.google.com/container-registry/docs/get-image-vulnerabilities)
*   [二进制授权](https://cloud.google.com/binary-authorization/)
*   [gVisor 容器沙箱](https://cloud.google.com/blog/products/gcp/open-sourcing-gvisor-a-sandboxed-container-runtime)
*   [云安全扫描仪](https://cloud.google.com/security-scanner/)
*   [云安全指挥中心](https://cloud.google.com/security-command-center/)

阅读以下内容以了解:

*   [保护和审计谷歌云平台边界](/google-cloud/secure-and-audit-the-google-cloud-platform-perimeter-a3c33f451a82)
*   [安全运营中心数据湖](https://medium.com/p/4b31e011f622/edit)

# 承认

非常感谢张世铭和耐莉·波特使这篇文章变得更好。