# 用 Terraform 管理云 SQL 资源

> 原文：<https://medium.com/google-cloud/managing-cloud-sql-resources-with-terraform-76cc044319e9?source=collection_archive---------0----------------------->

![](img/fd9c67d0022940ba5e0a5559f387bf65.png)

马库斯·斯皮斯克在 [Unsplash](https://unsplash.com/s/photos/code?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上的照片

很长一段时间以来，Terraform 在我的生活中占有特殊的位置，那就是著名乐队[虫胶](https://en.wikipedia.org/wiki/Shellac_(band))的专辑。HashiCorp 创造的基础设施作为代码软件工具，我已经知道，但从未真正使用过。

因此，在本文中，我们将看看我第一次使用 Terraform 的经历，以及我们如何使用它来管理云 SQL 资源。为了理解，我假设你有一个…