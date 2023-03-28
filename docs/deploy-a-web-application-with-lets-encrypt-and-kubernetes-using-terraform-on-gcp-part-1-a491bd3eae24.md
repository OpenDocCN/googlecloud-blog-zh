# 在 GCP 上使用 Terraform 部署一个带有 Let's Encrypt 和 Kubernetes 的 Web 应用程序(第 1 部分)

> 原文：<https://medium.com/google-cloud/deploy-a-web-application-with-lets-encrypt-and-kubernetes-using-terraform-on-gcp-part-1-a491bd3eae24?source=collection_archive---------0----------------------->

# 介绍

在[上一篇文章](/google-cloud/quickly-deploy-applications-using-terraform-with-kubernetes-on-gcp-6a4d7d142839?sk=b6a9176314472e1fbdc5353fc401b537)中，我们学习了如何使用 Terraform 部署两个应用程序——WordPress 和 Guestbook。然而，这两个应用程序已经被其他人很好地开发了。这一次，我将展示如何部署一个我们自己制作的实际应用程序。这意味着我们能够看到应用程序的源代码。我将整个事情分为两部分:第一部分，这篇文章，将讨论如何部署应用程序与让我们在 GCP 加密…