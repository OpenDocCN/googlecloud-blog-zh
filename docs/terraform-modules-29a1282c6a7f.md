# 地形模块

> 原文：<https://medium.com/google-cloud/terraform-modules-29a1282c6a7f?source=collection_archive---------2----------------------->

本文描述了 Terraform 中模块的使用和好处，以及它的实现。

Terraform 使用模块实现了 DRY(不要重复自己)原则，通过它可以对资源集合进行分组，以便以后重用。模块与编程语言中用于避免代码重复的函数非常相似。

**为什么要在 Terraform 中使用模块？**

*   **可读**:通过调用源模块消除代码行。
*   **可重用**:模块可以一次编写，多次重用。
*   **摘要**:将配置分离成逻辑单元。

在本文结束时，您将能够:

*   定义/创建模块。
*   使用模块重用配置
*   使用输入变量来参数化您的配置和输出，以将资源属性传递到模块外部。

**1。在地形中定义模块:**

Terraform 预装在云壳中。通过在云 shell 中运行 *terraform* 来验证 Terraform 是否可用。您也可以通过运行 *terraform — version 来检查 Terraform 的版本。*

1.  创建 terraform 文件的目录。
2.  每个目录都有自己的 *main.tf* 文件。
3.  在 *main.tf* 文件中编写代码来创建您的需求。

例如，如果您想要创建一个网络配置文件，创建一个 *network.tf* 文件并编写以下代码。

```
# Create network
resource "google_compute_network" "net1" {
name = "net1"
auto_create_subnetworks = "true"
}
# Add a firewall rule on net1
resource "google_compute_firewall" "net1-allow-http-ssh-rdp-icmp" {
name = "net1-allow-http-ssh-rdp-icmp"
network = google_compute_network.net1.self_link
allow {
protocol = "tcp"
ports = ["22", "80", "3389"]
}
allow {
protocol = "icmp"
}
source_ranges = ["0.0.0.0/0"] 
```

**2。使用模块复用配置:**

若要使用模块，请调用模块来引用模块块中的代码。

父模块使用 source 参数调用已定义的模块。运行 *terraform init* 命令下载配置引用的任何模块。指定源参数，即定义模块源位置的强制元参数。

*模块来源:*

**本地路径:**

用于引用与调用模块存储在同一目录中的模块。

```
module "web-server"{
source = ./server
}
```

**使用公共注册表中的模块:**

***地形注册表:***

它包含各种基础设施模块的公共可用模块的注册表。

来源名称格式:*名称空间/名称/提供者*

***Github 注册表:***

要使用这个，直接输入源代码所在的 Github URL。

运行命令 *terraform init* 安装提供者二进制文件并初始化新的 terraform 配置。一旦所有的代码都被写入各自的文件，现在通过运行命令 *terraform apply，*来应用您的配置。

**3。使用输入变量来参数化您的配置和输出，以将资源属性传递到模块外部。**

变量帮助您在不改变源代码的情况下定制模块的各个方面。它们在处理不同的环境(如生产、开发和试运行)时非常有用。

1.  用变量替换模块中的硬编码值。

```
resource "google_compute_network" "net1"{
name = var.<variable_name>
}
```

2.创建一个文件 *variable.tf* ，并在文件中声明变量。

```
variable "<variable_name>" {
type = string
description = "description about the variable"
}
```

3.调用模块时，传递输入变量的值。

```
module "prod_network"{
source = ./network
<variable_name> = "mynet1"
}
```

(将 *< variable_name >* 替换为您想要赋予变量的名称。)

在运行时向变量传递值是不可能的。因此，为了将资源参数从一个模块传递到另一个模块，参数被配置为 Terraform 中的输出值。

1.  若要从模块传递属性，请将属性的名称声明为输出值，以便在模块外部公开它。
2.  将该属性名定义为需要使用该值的模块中的变量。
3.  当调用需要使用该值的模块时，请参考输出值。

*注意:*可以使用格式*模块引用输出值。<模块名称>。<输出值>。*

**结论:**

在 Terraform 中，模块的概念用于重用配置，并编写有效、可重用和可读的代码。此外，考虑编写模块的最佳实践。