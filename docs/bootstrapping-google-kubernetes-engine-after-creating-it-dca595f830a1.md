# 创建 Google Kubernetes 引擎后引导它

> 原文：<https://medium.com/google-cloud/bootstrapping-google-kubernetes-engine-after-creating-it-dca595f830a1?source=collection_archive---------0----------------------->

我遇到了一个相当恼人的问题:**在我通过 Terraform 在 Google Cloud 上创建了**我的托管 **Kubernetes 集群**之后，**我想在同一次运行中为它们提供一些默认设置**。因此，如果我重新部署整个集群，我不想手动运行`kubectl apply`或创建我的名称空间、应用我的 RBAC 规则等的不同 CI 管道。

这被证明是一个相当令人沮丧的任务，尝试使用`local-exec`、`Terraform in Terraform`提供者、嵌套`gcloud`命令来获取 GKE 凭证、手工构建定制的`kubeconfig`文件，等等。最后，我找到了一个非常干净漂亮的解决方案，让我与你分享。

![](img/0889c4e38c63e4463dcbcd6020185c79.png)

当你意识到它其实有多简单的时候！(照片由 [Mubariz Mehdizadeh](https://unsplash.com/@mehdizadeh?utm_source=medium&utm_medium=referral) 在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄)

该解决方案实际上隐藏在 Terraform Google provider 网站的一个数据源[示例](https://www.terraform.io/docs/providers/google/d/datasource_client_config.html)中。

# 使用官方 K8s 提供商

简而言之:

*   您可以动态地引用您的`cluster’s IP address and CA certificate`(这将很好地处理资源依赖性，例如，首先创建您的集群，然后尝试在其上应用东西)
*   您可以查询您的`gcloud OAuth token`(这似乎是所有其他解决方案中最难发明的一个)

```
# Query my Terraform service account from GCP
data "google_client_config" "current" {}provider "kubernetes" {
  load_config_file = false host = "[https://${module.gke_cluster.endpoint](https://${module.cluster_gke1.endpoint)}"
  cluster_ca_certificate = "${base64decode(module.gke_cluster.cluster_ca_certificate)}" token = "${data.google_client_config.current.access_token}"
}
```

然后，您就可以调配资源了！

```
resource "kubernetes_namespace" "test" {
  metadata {
    name = "test"
  }
}
```

# 同一 Terraform 工作流中的多个集群

那么，如果您创建了多个 GKE 集群会怎么样呢？[提供商化名](https://www.terraform.io/docs/configuration/providers.html#multiple-provider-instances)前来救援！

```
provider "kubernetes" {
  **alias = "gke1"** load_config_file = false host = "[https://${module.gke_cluster1.endpoint](https://${module.cluster_gke1.endpoint)}"
  cluster_ca_certificate = "${base64decode(module.gke_cluster1.cluster_ca_certificate)}" token = "${data.google_client_config.current.access_token}"
}
```

当您调用一个资源时，您只需添加一个额外的`provider`行:

```
resource "kubernetes_namespace" "test" {
  **provider = "kubernetes.gke1"** metadata {
    name = "test"
  }
}
```

…也许这就是你意识到官方的 Kubernetes 供应商是多么缺乏维护的地方。

它几乎没有任何你可能想要提供的有用资源。仅举几个例子:角色绑定、集群角色、部署、守护进程、入口…基本上只有**重要的东西没有**！:D

# 使用社区维护的 K8s 提供程序

很幸运，我撞上了官方插件的一个惊人分叉:[https://github.com/sl1pm4t/terraform-provider-kubernetes](https://github.com/sl1pm4t/terraform-provider-kubernetes)

设置是相当棘手的，因为你实际上必须手动包含提供者二进制文件，但是你仍然需要从互联网上下载其他插件。当你使用定制插件时，这工作得相当完美。但是！

> 当自定义插件与官方插件同名时……Terraform 将总是下载官方插件，而完全忽略您的自定义二进制文件。

```
provider "kubernetes" {
  # We use a custom plugin here because the official is very outdated. Please note the -custom suffix, it's important!
  # Download from [https://github.com/sl1pm4t/terraform-provider-kubernetes/releases](https://github.com/sl1pm4t/terraform-provider-kubernetes/releases) and **unzip to** the current
  # TF workspace under **terraform.d/plugins/<your architecture>** (darwin_amd64, linux_amd64)
  **version =** "1.2.0**-custom**" load_config_file = false host = "[https://${module.cluster_gke1.endpoint](https://${module.cluster_gke1.endpoint)}"
  cluster_ca_certificate = "${base64decode(module.cluster_gke1.cluster_ca_certificate)}" token = "${data.google_client_config.current.access_token}"
}
```

## 插件自动发现的诀窍

所以这里的魔法只是

*   下载并解压缩定制的二进制文件
*   让 Terraform **自动发现**你的插件

如果你遵循 GitHub 文档并在`terraform init -plugin-dir=<custom plugin folder>`之前进行**手动发现**，它**将会破解**所有需要下载官方提供商的东西，例如谷歌云提供商。

因此，你可以做的是:在你当前的 Terraform 工作目录*中创建一个`terraform.d/plugins/darwin_amd64/`(或`linux_amd64`)文件夹(别忘了把它放进去。gitignore)* 并在那里解压插件。

现在初始化 Kubernetes 提供者将会完全忽略你的文件夹并下载官方的…这是因为他们有相同的名字。即使你把`version ~> 1.12`，因为他们现在都有相同的发布版本，将默认为官方。

最后一点提示:**给你请求的插件版本添加** `**-custom**` **后缀。这将确保它回落到你自己的二进制。(相反，缺点是你将来必须手动更新插件。)**

使用 Terraform，您可以在 Kubernetes 中做更多的事情:

```
resource "kubernetes_cluster_role_binding" "admin" {
  # docs: [https://github.com/sl1pm4t/terraform-provider-kubernetes/pull/39](https://github.com/sl1pm4t/terraform-provider-kubernetes/pull/39)
  metadata {
    name = "tf-admin"
  } role_ref {
    name  = "cluster-admin"
    kind  = "ClusterRole"
  }
  subject {
    kind = "User" # this is actually my TF service account in GCP
    name = "[terraform@<my gcp project>.iam.gserviceaccount.com](mailto:terraform-lab@edo-tf-resources.iam.gserviceaccount.com)"
  }
}
```

在撰写本文时，这个提供者的一个缺点是:*实际上没有文档*——您要么阅读语法代码，要么浏览 GitHub 问题/拉请求。都在那里，只需要做一些额外的回合。

希望你发现这个快速指南至少对我有用！如果您在创建集群的同时发现了其他动态获取 Kubernetes 凭证的解决方案，请在下面的评论中告诉我。