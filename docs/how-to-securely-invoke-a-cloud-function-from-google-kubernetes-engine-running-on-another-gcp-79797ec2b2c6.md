# 如何从运行在另一个 GCP 项目上的 Google Kubernetes 引擎安全地调用云函数

> 原文：<https://medium.com/google-cloud/how-to-securely-invoke-a-cloud-function-from-google-kubernetes-engine-running-on-another-gcp-79797ec2b2c6?source=collection_archive---------0----------------------->

在复杂的环境中，不同的团队运行他们自己的 Google Cloud 项目，很难确保一个项目中的服务只能被运行在其他 Google Cloud 项目上的特定应用程序访问。复杂的 VPC 对等和内部负载平衡方案常常是不可避免的，有时甚至不可能在不将服务暴露给涉及多个区域的公共互联网的情况下实现跨项目通信。