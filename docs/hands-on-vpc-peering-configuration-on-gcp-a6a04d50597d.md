# GCP 上的 VPC 对等配置实践

> 原文：<https://medium.com/google-cloud/hands-on-vpc-peering-configuration-on-gcp-a6a04d50597d?source=collection_archive---------1----------------------->

# **简介**

谈到 GCP 网络，我们必须知道什么是虚拟私有云(VPC)。根据 GCP 文档，虚拟专用云(VPC)网络是物理网络的虚拟版本，例如数据中心网络。它为您的[计算引擎虚拟机(VM)实例](https://cloud.google.com/compute/docs/instances)、 [Google Kubernetes 引擎(GKE)集群](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture)、 [App Engine 灵活环境](https://cloud.google.com/appengine/docs/flexible)实例以及您的[项目](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy#projects)中的其他资源提供连接。GCP 有一个很棒的关于 VPC 的视频[真的推荐大家去看。从我的观点来看…](https://www.youtube.com/watch?v=cNb7xKyya5c)