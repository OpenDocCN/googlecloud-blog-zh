# 使用 VPC 服务控件设计安全的数据管道

> 原文：<https://medium.com/google-cloud/designing-secure-data-pipelines-with-vpc-service-controls-e3b4502307df?source=collection_archive---------0----------------------->

如今，GCP 有很多数据分析用例。在 GCP 建立数据分析平台最常见的问题之一是安全性。谷歌云有很多安全产品来保护客户的数据，其中之一是 VPC 服务控制(VPC SC)。

VPC SC 为谷歌云服务提供了一个独立于身份和访问管理(IAM)的额外安全防御层。IAM 支持精细的基于身份的访问控制，而 VPC SC 支持更广泛的基于上下文的外围安全…