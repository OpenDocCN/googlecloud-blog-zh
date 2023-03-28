# GCP CIS 基准 Terraform 模块测试与 Chef Inspec、Kitchen-terra form & GitHub Actions—第 1 部分

> 原文：<https://medium.com/google-cloud/gcp-cis-benchmark-terraform-module-testing-with-chef-inspec-kitchen-terraform-github-actions-d82dabde8290?source=collection_archive---------0----------------------->

![](img/44550e83c74662efe6df9c145ba89e58.png)

我开始摆弄[厨房平台](https://github.com/newcontext-oss/kitchen-terraform)，让[厨师检查](https://www.chef.io/products/chef-inspec)我的平台模块的测试和符合性。我目前使用的是 [inspec-gcp](https://github.com/inspec/inspec-gcp) 资源包和 [inspec-gcp-cis-benchmark](https://github.com/GoogleCloudPlatform/inspec-gcp-cis-benchmark) 概要文件。本文将重点讨论后者。这样做的驱动力是测试代码的不言而喻的好处，并把一些合理的基本安全标准转移到左边。

为了简单起见，我想从你开始在 Google 中构建云基础设施时首先要做的事情开始。任何谷歌云平台服务的基础都是一个[项目](https://cloud.google.com/resource-manager/docs/creating-managing-projects)。我要讲的模块是[这里是](https://github.com/lzysh/terraform-google-project)供参考，涵盖了构建项目。这是一个合理的自以为是的 terraform 模块，适合我的一些用例；然而，我认为它展示了我正在尝试做的事情的例子，而不会太复杂和令人困惑。因为老实说，这对于像我这样的非开发人员、前运营思维的人来说很难。

*注意:我不会深入介绍设置 Terraform、Ruby、Kitchen-Terraform、Chef Inspec 等的基础知识。如果那是有趣的，让我知道，也许我能做另外一个职位。*

# **“本地”开发:**

从软件开发的实践中学习，我们需要能够“本地”开发。当我在这里说本地时，我们可以认为它是谷歌云平台[资源层级](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy)中的一个沙箱区域，在这里你可以安全地创建和构建基础设施。当您开始开发 Terraform [模块](https://www.terraform.io/docs/configuration/modules.html)时，您很可能会在 main.tf 中看到类似这样的内容:

```
# Project Resource
# [https://www.terraform.io/docs/providers/google/r/google_project.html](https://www.terraform.io/docs/providers/google/r/google_project.html)resource "google_project" "this" {
  name                = var.project_id
  project_id          = var.project_id
  billing_account     = var.billing_id
  folder_id           = "folders/${var.folder_id}"
}
```

和一个 variables.tf 文件，如下所示:

```
variable "billing_id" {
  description = "Billing ID for the project to use"
  type        = string
}variable "project_id" {
  description = "Project ID (This will be used for the project name as well)"
  type        = string
}variable "folder_id" {
  description = "Folder ID for the project to be created in."
  type        = string
}
```

我们现在可以运行 terraform 并创建我们的谷歌云项目:

```
terraform initterraform plan -out plan.out \
-var=”billing_id=00000C-AZAZAZ-EFEFEF” \
-var=”project_id=test-del-me-4876des” \
-var=”folder_id=993877078800"terraform apply plan.out
```

现在你已经启动并运行了一个项目，但是你可能会惊讶于你已经违反了一堆 [CIS 基准](https://www.cisecurity.org/cis-benchmarks/)！感谢谷歌和人们在[inspec-GCP-cis-benchmark](https://github.com/GoogleCloudPlatform/inspec-gcp-cis-benchmark)GitHub 知识库中所做的工作；我们可以测试一下！

*注意:这不是一个官方支持的谷歌产品，但希望他们在社区的帮助下维护这个回购。*

我喜欢这个 GitHub 项目的第一点是，它直接针对一个特定的谷歌云项目运行。对我来说，这正是我想要运行测试的级别。稍后，我将详细介绍这一点，但是现在，让我们检查一下我们刚刚构建的项目，看看我们需要做什么:

```
inspec exec [https://github.com/GoogleCloudPlatform/inspec-gcp-cis-benchmark.git](https://github.com/GoogleCloudPlatform/inspec-gcp-cis-benchmark.git) \
-t gcp:// — input gcp_project_id=test-del-me-4876des
```

运行该命令后，您将看到一个违规列表。本着测试驱动开发(TDD)的精神，我们有失败的测试，现在我们可以编码来修复它。我们将在下一篇文章中整合这一切。为了简单起见，让我们集中讨论其中的两个:

```
× cis-gcp-4.4-vms: [VMS] Ensure oslogin is enabled for a Project
× cis-gcp-3.1-networking: [NETWORKING] Ensure the default network does not exist in a project
```

这里的想法是，我们在过程中尽可能远地解决这些问题。我仍在“本地”开发如果你在默认网络上部署了一堆东西，三年后，你的安全团队说，我们需要让所有东西都符合 CIS 你将会有难以控制的操作工作量！现在让我们解决这些合规性问题。为此，我们可以在 Terraform 模块代码中添加几行代码。然后，您组织中使用它的每个人都将符合标准。

```
# Project Resource
# [https://www.terraform.io/docs/providers/google/r/google_project.html](https://www.terraform.io/docs/providers/google/r/google_project.html)resource "google_project" "this" {
  name                = var.project_id
  project_id          = var.project_id
  billing_account     = var.billing_id
  folder_id           = "folders/${var.folder_id}"
  auto_create_network = false
}# Project Metadata Resource
# [https://www.terraform.io/docs/providers/google/r/compute_project_metadata.html](https://www.terraform.io/docs/providers/google/r/compute_project_metadata.html)resource "google_compute_project_metadata" "this" {
  project = google_project.this.project_id
  metadata = {
    enable-oslogin = true
  }
}
```

接下来，您可以销毁之前的项目，并使用下面的代码重新创建它，您将看到您已经通过了测试:

```
✔ cis-gcp-3.1-networking: [NETWORKING] Ensure the default network does not exist in a project
✔ cis-gcp-4.4-vms: [VMS] Ensure oslogin is enabled for a Project
```

不仅如此，我们还从:

```
Profile Summary: 4 successful controls, 16 control failures, 21 controls skippedTest Summary: 12 successful, 40 failures, 79 skipped
```

收件人:

```
Profile Summary: 5 successful controls, 11 control failures, 22 controls skipped Test Summary: 7 successful, 12 failures, 80 skipped
```

我觉得结果不言自明。在下一篇文章中，我们将关注厨房平台。