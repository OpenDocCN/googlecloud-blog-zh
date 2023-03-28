# 具有自签名证书的 GKE 和私人注册中心

> 原文：<https://medium.com/google-cloud/gke-and-private-registries-with-self-signed-certificates-b37b5fd1f982?source=collection_archive---------0----------------------->

**注意:本文提供了一种变通方法，这不是 Google Cloud 官方文档程序，使用风险自担，并且只能在非生产系统上使用**

# **我们试图解决什么问题？**

GKE 无法从使用未由可信的 [CA](https://en.wikipedia.org/wiki/Certificate_authority) 签名的证书的注册表中提取图像:如果节点上的 [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) 无法验证它试图从中提取图像的注册表的 CA 授权，应用程序窗格将因错误而停滞在容器创建阶段…