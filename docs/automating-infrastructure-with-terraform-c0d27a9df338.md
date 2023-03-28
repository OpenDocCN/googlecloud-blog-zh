# 使用 Terraform 实现基础设施自动化

> 原文：<https://medium.com/google-cloud/automating-infrastructure-with-terraform-c0d27a9df338?source=collection_archive---------0----------------------->

这篇文章进一步详细介绍了在 Google Cloud 上使用 Terraform 作为代码来管理基础设施。与通过用户界面手动配置资源不同，基础结构作为代码涉及控制一个或多个文件中的基础结构。Terraform 是 HashiCorp 提供的基础设施代码。它是一个安全、一致地开发、改变和管理基础设施的工具。

先决条件:熟悉谷歌云控制台，虚拟机，网络配置。

在本文结束时，您将能够:

*   使用 Terraform 建立、变更和摧毁基础设施
*   使用 Terraform 创建资源依赖关系
*   使用 Terraform 调配基础架构

**1。建筑基础设施**

Terraform 预装在云壳中。通过在云壳中运行 *terraform* 来验证 Terraform 是否可用。

> 用这段代码创建 main.tf 文件(替换<peroject id="" with="" your="" project="">)。这是描述运行所需组件的配置文件。</peroject>

```
terraform {
  required_providers {
    google = {
      source = "hashicorp/google"
    }
  }
}
provider "google" {
  version = "3.5.0"
  project = "<PROJECT_ID>"
  region  = "us-central1"
  zone    = "us-central1-c"
}
resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
```

terraform {}块是必需的，因此 terraform 知道从 [Terraform 注册表](https://registry.terraform.io/)下载哪个提供者。资源块在打开块之前有两个字符串:资源类型(Google _ compute _ network**)**和资源名称( vpc_network **)** 。

> *运行命令* terraform init *安装提供者二进制文件并初始化新的 terraform 配置。*
> 
> *运行命令* terraform apply，立即应用您的配置。

输出在资源“Google _ compute _ network”“VPC _ network”旁边有一个+,意味着 terraform 将创建这个资源。

如果计划创建成功，terraform 将暂停并等待批准，然后继续。如果 terraform 应用因错误而失败，请阅读错误消息并修复出现的错误。

仅此而已，terraform 全部搞定！您可以转到云控制台，看到虚拟机实例已创建，网络已调配。要么，在云壳中运行 *terraform show* 命令来检查当前状态。

**2。改变和破坏基础设施**

您可以通过将资源添加到 terraform 配置中并运行 terraform apply 来供应资源，从而创建新资源。

向“vm_instance”添加一个 tags 参数，如下所示:

```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  # ...
}
```

要更新实例，再次运行 *terraform apply* 。如果前缀存在，资源将被就地更新。您可以继续并接受修改。

破坏性更改是指提供者必须替换当前资源，而不是简单地更新它。通常，这是因为云提供商不支持根据您的设置升级资源。如果前缀-/+存在，资源将被销毁并重新创建，而不是就地更新。

在生产环境中，基础设施破坏很少发生。但是，如果您使用 terraform 来加速各种环境，如开发、测试和登台，销毁通常是一个有用的操作。类似于 *terraform apply* 的 *terraform destroy* 命令可用于销毁资源，但其操作方式如同所有资源都已从配置中移除。使用前缀-时，网络和实例将被终止。在这里，它创建了一个依赖图来标识删除资源的顺序。

**3。创建资源依赖关系**

现实世界中的基础设施使用各种资源。terraform 配置中可以包含多种资源、不同的资源种类，甚至不同的提供者。

在这里，我们将看到如何配置多个资源以及如何使用资源属性来配置其他资源的基本示例。

将此块添加到您的配置文件中，以便为 main.tf 中的 VM 实例分配一个静态 IP:

```
resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
```

当 Terraform 读取此配置时，它将:

*   确保在虚拟机实例之前创建虚拟机静态 ip
*   在状态中保存 vm_static_ip 的属性
*   将 nat_ip 设置为 vm_static_ip.address 属性的值

> *再次运行 terraform 计划，并使用* terraform 计划输出 static_ip *保存该配置。*

以这种方式保存计划可以确保您将来可以应用完全相同的计划。在上面的例子中，对 google_compute_address . vm_static_ip . address 的引用创建了对名为 VM _ static _ IP 的 Google _ compute _ address 的隐式依赖。

通过插值表达式的隐式依赖是通知 Terraform 这些关系的主要方式，应尽可能使用。有时资源之间的依赖关系对 terraform 来说是不可见的。 *depends_on* 参数可以添加到任何资源中，并接受一个资源列表来为其创建显式依赖关系。

**4。供应基础设施**

谷歌云允许客户管理他们自己的[定制操作系统映像](https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images)。这是一个很好的方法，可以确保您使用 Terraform 提供的实例是根据您的需求预先配置的。 [Packer](https://www.packer.io/) 是这方面的完美工具，包括一个用于谷歌云的[构建器。](https://www.packer.io/docs/builders/googlecompute.html)

Terraform 使用供应器来上传文件、运行 shell 脚本或安装和触发其他软件，如配置管理工具。

```
resource "google_compute_instance" "vm_instance" {
  name         = "terraform-instance"
  machine_type = "f1-micro"
  tags         = ["web", "dev"]
  provisioner "local-exec" {
    command = "echo ${google_compute_instance.vm_instance.name}:  ${google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
  }
  # ...
}
```

这里使用的 *local-exec* provisioner 在运行 terraform 的机器上本地执行命令，而不是 VM 实例本身。

Terraform 对置备程序的处理不同于其他参数。置备程序仅在创建资源时运行，但添加置备程序不会强制销毁和重新创建该资源。使用*terraform tain*命令告诉 terra form 重新创建实例。被污染的资源将在下次应用时被销毁并重新创建。

**结论**

terraform 的使用使得管理和更新基础设施的任务变得相当简单。更进一步，你可以使用谷歌云持续集成服务 [Cloud Build](https://cloud.google.com/build) ，自动将 terraform 清单应用到你的环境中。