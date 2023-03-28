# 如何使用 VPC 服务控件保护您的 Apigee X 数据

> 原文：<https://medium.com/google-cloud/how-to-secure-your-apigee-x-data-with-vpc-service-control-ea30b2303790?source=collection_archive---------0----------------------->

# TL；速度三角形定位法(dead reckoning)

**目标**:这篇文章提供了用 GCP VPC 服务控件保护您的 Apigee X 部署数据的必要步骤列表。

*虽然这意味着一步一步的指导，但我建议通过将命令与 Terraform 或任何基础设施集成为代码软件工具来自动化该过程。*

# 语境

越来越多的公司每天都面临着越来越多的**安全风险**,这些风险与他们的数据通过不同途径暴露有关…