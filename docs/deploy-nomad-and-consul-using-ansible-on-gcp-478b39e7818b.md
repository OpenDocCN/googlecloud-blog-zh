# 在 GCP 使用 Ansible 部署游牧民和领事

> 原文：<https://medium.com/google-cloud/deploy-nomad-and-consul-using-ansible-on-gcp-478b39e7818b?source=collection_archive---------0----------------------->

# 介绍

在我之前的[文章](/google-cloud/introduction-to-docker-and-kubernets-on-gcp-with-hands-on-configuration-part-1-docker-3d9709ee9f6a?sk=1795abde6f7cfb42f1c05693e6ce99e0)中，我们谈到了使用容器技术的优势。而最著名的容器编排工具应该是 Kubernetes (k8s)。但是 k8s 不好维护，而且相当复杂。你可能会问的下一个问题是，有简化的容器编排吗？是的，诺曼德。如果你没有听说过 Nomad，你可能听说过 Terraform，事实上他们都在同一家名为 HashiCorp 的公司旗下。

# 流浪者