# 在 GCP 上使用带有 Kubernetes 的 Terraform 快速部署应用程序

> 原文：<https://medium.com/google-cloud/quickly-deploy-applications-using-terraform-with-kubernetes-on-gcp-6a4d7d142839?source=collection_archive---------1----------------------->

# 介绍

在之前的文章中，我们学习了如何使用 Kubernetes 部署 Wordpress。尽管在那篇文章中，我已经准备了一些脚本来部署，而且已经足够快了，但是还有一种方法可以部署得更快。是的，我们可以使用 Terraform。如果你有 AWS 的经验，你可能听说过 CloudFormation。Terraform 类似于 CloudFormation，但它可以应用于不同的云提供商。**在本文中，我们将逐步使用 Terraform 部署两个应用程序**。