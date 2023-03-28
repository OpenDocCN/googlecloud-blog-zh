# GCP 的 Terraform 凭据设置

> 原文：<https://medium.com/google-cloud/terraform-credentials-setup-in-gcp-c81c8ebaff5d?source=collection_archive---------0----------------------->

如何在 Google 云平台中创建一个 terraform service-account，以及如何在本地生成和使用其凭证。

![](img/3df2d2724af6c3c95aac4d6d6f6ac0e6.png)

Google Cloud IAM + Terraform 徽标

我们需要向 GCP 认证才能使用 terraform。根据[谷歌云平台文档](https://cloud.google.com/community/tutorials/managing-gcp-projects-with-terraform)，推荐的做法是为 terraform 创建一个服务帐户，并为其提供创建基础设施所需的访问权限。