# 资源工厂:地形的描述方法

> 原文：<https://medium.com/google-cloud/resource-factories-a-descriptive-approach-to-terraform-581b3ebb59c?source=collection_archive---------1----------------------->

![](img/32e3428cfc1c6d697d3d88c1ae82992b.png)

[](https://www.flickr.com/photos/24736216@N07/2994043188)[Roger 4336](https://www.flickr.com/photos/24736216@N07)的【沃尔夫斯堡—大众装配线】获得 [CC BY-SA 2.0](https://creativecommons.org/licenses/by-sa/2.0/?ref=ccsearch&atype=rich) 的许可

# 序文

随着您的 Terraform 代码库的大小和范围的增长，您可能会发现自己处于这些常见场景之一:

*   **启用特定团队的贡献**(例如，网络运营团队创建新的子网或防火墙规则，或者日志和监控团队配置新的警报)**可能很困难**，因为 Terraform 知识在团队中并不普及
*   给定种类的资源(例如防火墙规则、子网、项目)的重复创建分散在整个代码库中，这使得清楚地了解它们的结构变得很复杂
*   坚持对大型基础设施采用单一的端到端方法(即以大型模块/状态描述整个基础设施)会导致**更慢、更难维护的代码库**，这并不能反映基础设施发展的不同速度(想想核心基础设施与防火墙规则)

本文提出了一种基于配置的方法，这种方法能够将特定资源的重复创建分配到不同的存储库中，并使不支持 Terraform 的团队能够为您的 IaC 代码库做出贡献。

# 传统方法

让我们看一个例子，看看我们如何使用传统范围内的解决方案来处理相同种类的多个资源的创建。

**使用提供商内置资源的幼稚定义。**没有解决让不同团队为代码库做出贡献的挑战，当资源数量达到几十或几百时，可能会变成冗长且难以阅读的代码墙，并且通常不适合生产使用。

**将逻辑包装在 Terraform 模块中，并利用 Terraform 变量。**这种方法允许代码和配置的分离，允许非核心团队向特定的存储库贡献代码，并为他们提供一个更容易使用的界面(配置与地形代码)。然而，随着资源数量的增长，你的变量很快变得更加难以解析和理解，一个小小的输入错误就可能导致难以解决的错误。

# 描述性方法

**在 Terraform 资源工厂中包装逻辑，并利用 YaML/JSON 配置，每个资源一个。**这是我们在大型复杂代码库上使用的久经考验的方法。

一个**资源工厂**是一个**固执己见、专门构建的**模块

*   **实施**特定的**要求和最佳实践**(例如，“始终为谷歌云 VPC 子网启用 [PGA](https://cloud.google.com/vpc/docs/configure-private-google-access) ”或“仅允许使用谷歌云区域欧洲-西方 1 和欧洲-西方 3”)
*   **编纂业务逻辑和政策**(如标签和命名约定)
*   **标准化、自动化和集中化**给定类型资源的重复创建

我们的方法基于使用 Terraform 代码实现工厂逻辑的模块，以及一组具有良好定义的语义结构的目录，以 YaML 语法保存资源的配置。

**Terraform 本身支持 YaML、JSON 和 CSV** 解析——一些观察:

*   YaML 对人来说更容易解析，并且允许注释和嵌套的复杂结构
*   JSON 和 CSV 不能包含注释，注释可以用来记录配置，但通常对于在自动化管道中从其他系统桥接过来很有用
*   JSON 更冗长(读起来更长),也更难被人解析
*   CSV 通常不够有表现力(例如，不支持嵌套结构)

**Tl；dr，如果你是手工编辑文件，YaML 可能是你最好的选择。**

记住这一点，让我们看一个使用[资源工厂](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/tree/master/blueprints/factories)部分的[云基础架构](https://github.com/terraform-google-modules/cloud-foundation-fabric)中可用的`subnets`模块的方法示例。

该模块被设计为只需将它指向我们的配置文件夹即可使用

```
module "subnets" {
  source        = "./cloud-foundation-fabric/factories/subnets"
  config_folder = "./subnets"
}
```

…使用固定的数据目录结构

```
└── subnets             # Configuration folder entry point
  ├── project-ada       # Project ID the VPC belongs to 
  │ ├── vpc-alpha 
  │ │ ├── subnet-a.yaml # Subnet name 
  │ │ └── subnet-b.yaml 
  │ └── vpc-beta 
  │   └── subnet-c.yaml 
  └── project-bob
    └── vpc-gamma
      └── subnet-d.yaml
```

…以及用于实际配置的 YaML 文件，每个子网一个文件

```
# subnets/project-ada/vpc-alpha/subnet-a.yaml 
region: europe-west1 
description: Frontend 
ip_cidr_range: 10.0.0.0/24 
secondary_ip_ranges: 
  secondary-range-a: 192.168.0.0/24
  secondary-range-b: 192.168.128.0/24# subnets/project-ada/vpc-alpha/subnet-b.yaml 
region: europe-west1 
description: Backend 
ip_cidr_range: 10.0.1.0/24 
# Assign roles/compute.networkUser 
iam_groups: ["backend-admins@example.com"]
```

**文件夹结构为每个子网提供了许多必需的属性**(项目 id、VPC 名称、子网名称)**，减少了出错的可能性**，并使网络结构**显式**和**易于导航**，甚至从存储库浏览器视图也是如此，而 YaML 文件结构是必不可少的，对于不熟悉 terraform 的操作员来说**易于理解和维护**，使他们能够为您的 IaC 代码库做出贡献。

**然后可以实现 GitOps 策略**来检查每个合适的提交的**正确性**(例如，YaML 语法、`terraform plan`输出)和**符合性**(例如，仅允许某些用户/组在给定的文件夹下提交)，并最终`terraform apply`得到基础设施更新。

这只是一个例子，说明在管理大型复杂的基础设施时，资源工厂是多么方便。除了子网，[我们的代码库](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/tree/master/blueprints/factories)还有越来越多的其他工厂(分级防火墙策略、VPC 防火墙规则)，您可以从中获得灵感，或者直接获取并适应您的需求。