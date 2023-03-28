# 如何为内部流量全局公开 Apigee

> 原文：<https://medium.com/google-cloud/how-to-globally-expose-apigee-for-internal-traffic-bd6c524709d6?source=collection_archive---------0----------------------->

# TL:DR

本文提供了一个分步指南，介绍如何利用 GCP 内部负载均衡器的**“全局访问”特性，在您的组织内** **跨多个地区**公开不同的 API 服务。正如您将看到的，端到端架构利用 [Apigee](https://cloud.google.com/apigee) 作为后端和负载平衡器之间的 API 网关，但它也适用于其他场景。

# 语境

在与许多客户打交道时，我很快意识到所有公司的第一步…