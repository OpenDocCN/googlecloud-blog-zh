# 保护和审计谷歌云平台边界

> 原文：<https://medium.com/google-cloud/secure-and-audit-the-google-cloud-platform-perimeter-685466c62d0f?source=collection_archive---------1----------------------->

## 护城河、城墙和城堡

你可能已经读过，[谷歌的 BeyondCorp 愿景](https://cloud.google.com/beyondcorp/)是…

> 一种企业安全模式，建立在 Google 6 年零信任网络的基础上，结合了来自社区的最佳理念和实践。通过将访问控制从网络边界转移到个人设备和用户，BeyondCorp 允许员工在任何位置更安全地工作，而不需要传统的 VPN。

[谷歌安全设计](http://services.google.com/fh/files/misc/csuite_security_ebook.pdf)的核心组成部分之一是…

> Google Cloud Platform 的基础设施安全性是按递进层次设计的——硬件、服务、用户身份、存储、互联网通信和运营。我们称之为深度防御。每一层都有严格的访问和权限控制。

然而，正如 [BeyondCorp 的研究论文](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43231.pdf)中所描述的，自 IT 基础设施的早期，企业就已经使用边界安全来保护和控制对内部资源的访问；这种模式很熟悉，对 B2B SaaS 客户尤其重要，对他们来说，谷歌云平台托管的 SaaS 应用程序看起来很像一个黑匣子。

[谷歌云的目标之一](https://cloud.google.com/why-google-cloud/)就是在你所在的地方与你相遇；本系列文章探讨了一些可用的控制措施，以保护 Google 云平台应用程序及其数据的边界，并验证您的信任。

# 外围安全和审计

传统的企业问题通常分为以下几个方面:

*   您如何保护用户向云的过渡？公共互联网上的 HTTPS 通常被认为不够安全。
*   您如何确保您的用户只与您或您的 SaaS 提供商的应用程序通信？企业通常通过仅支持与白名单 IP 的通信来保护自己的边界。
*   您如何审核流量和数据访问，也就是说，您如何知道控制是否按预期工作？

在以下章节中，我们将跨代表性的 Google 云平台服务依次探讨这些领域:

*   [计算引擎](https://cloud.google.com/compute/)，它与传统的企业虚拟机最为相似。
*   [Kubernetes](https://kubernetes.io/)/[Kubernetes 发动机](https://cloud.google.com/kubernetes-engine/)。
*   [App 引擎伸缩](https://cloud.google.com/appengine/docs/flexible/)。
*   [App 引擎标准](https://cloud.google.com/appengine/docs/standard/)。
*   [云存储](https://cloud.google.com/storage/)，作为托管数据服务的代表。

![](img/2bc33fd69b377b8641e92f4b15d2005c.png)

[标记为重复使用](https://www.google.com/search?lr=&hl=en&as_qdr=all&biw=1469&bih=1080&tbs=sur%3Afc&tbm=isch&sa=1&ei=-ISMW9_rKo3J0PEPvouokA8&q=fortress+keep+overhead+view&oq=fortress+keep+overhead+view&gs_l=img.3...1416.1416..1998...0.0..0.60.60.1......1....1..gws-wiz-img.NpUy_OU2eJc#imgdii=e8PSkbJlTBlWLM:&imgrc=NSRkXh7-UQkfjM:)

# 下一步是什么

阅读下面的内容，了解更多关于本文的基本概念:

*   [博彦科技](https://cloud.google.com/beyondcorp/)
*   [BeyondCorp 研究论文](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43231.pdf)
*   [纵深防御](https://paidpost.nytimes.com/google-cloud/defense-in-depth.html)
*   [安全电子书的谷歌云方案](http://services.google.com/fh/files/misc/csuite_security_ebook.pdf)

阅读以下指南，了解谷歌云平台在以下周边安全领域的能力。

*   [“私人”中转](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-d16372fa6697)
*   [受限 IP 范围](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-ade393c25467)
*   [审计](/@fargyle/secure-and-audit-the-google-cloud-platform-perimeter-a3c33f451a82)

# 承认

非常感谢 Ben Menasha 在这一系列文章中提供的所有智慧；任何错误都是我的。