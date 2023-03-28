# GKE 上 Knative v0.14.0 的工作负载身份认证

> 原文：<https://medium.com/google-cloud/workflow-identity-authentication-for-knative-v0-14-0-on-gke-9d6368a44c59?source=collection_archive---------1----------------------->

如果你曾经在谷歌云上使用过 Knative，你一定听说过 [Knative-GCP](https://github.com/google/knative-gcp) 项目。顾名思义，Knative-GCP 项目提供了许多来源，如`CloudPubSubSource`、`CloudStorageSource`、`CloudSchedulerSource`等，以帮助将各种谷歌云来源读入您的 Knative 集群。

我最近更新了我的 [Knative 教程](https://github.com/meteatamel/knative-tutorial)使用最新的 [Knative Eventing 版本 v0.14.2](https://github.com/knative/eventing/releases/tag/v0.14.2) 及其对应的 [Knative-GCP 版本 v0.14.0](https://github.com/google/knative-gcp/releases/tag/v0.14.0) 。我遇到了一个奇怪的认证问题，我想在这里概述一下。

最新的 Knative-GCP 版本中的一个显著差异是，现在它支持[工作负载身份](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)(除了现有的 Kubernetes Secret)作为控制和数据平面的认证机制(或者在文档中是这样声称的)。此外，文件指出，工作负载身份是从 GKE 访问谷歌云服务的推荐方式，因为它提高了安全性和可管理性。

当我按照[安装 GCP](https://github.com/google/knative-gcp/blob/master/docs/install/install-knative-gcp.md) 页面时，我无法让控制面板工作。原来在 [Knative-GCP 版本 v0.14.0](https://github.com/google/knative-gcp/releases/tag/v0.14.0) 中，工作负载标识仅**支持数据平面，而**不支持控制平面。****

此后，安装页面已更新为包括以下内容:

*选项 1(推荐):使用工作负载标识。注意:现在，控制平面的工作负载标识只有在您从主设备安装了 Knative-GCP 结构的情况下才有效。如果你用我们的最新版本(v0.14.0)或更早的版本安装了克奈夫-GCP 结构，请使用选项 2*

这基本上意味着您可以安装 Knative-GCP 的主版本，或者等待下一个版本，以获得适用于控制和数据平面的工作负载身份。或者，您可以只为数据平面设置工作负载身份，同时为控制平面使用 Kubernetes Secret 身份认证。

希望这能帮助一些人！

*原发布于*[*https://atamel . dev*](https://atamel.dev/posts/2020/05-20_workflow_identity_authentication_for_knative_on_gke/)*。*