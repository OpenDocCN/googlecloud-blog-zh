# 不要用 Terraform 部署应用程序

> 原文：<https://medium.com/google-cloud/dont-deploy-applications-with-terraform-2f4508a45987?source=collection_archive---------0----------------------->

![](img/1c392e49b663058929fb377602008a7e.png)

我知道。这是一个沉重的话题。我不敢相信我要去那里——深入一个如此热烈的话题，你可能已经有了强烈的观点。

说真的， [Terraform](https://www.terraform.io/) 不是应用部署软件。你们中的一些人已经以这种方式使用 Terraform。

天没有塌下来。是时候进行细致入微的讨论了。听我说完。

# 什么是 Terraform？

一般来说，Terraform 是一种云资源部署工具。Terraform 最常见的用例是管理云或虚拟资源的生命周期。你们中的许多人已经这样做了:你们正在启动[虚拟机](https://cloud.google.com/compute)，管理 [DNS](https://cloud.google.com/dns) 区域，创建[存储](https://cloud.google.com/storage)桶，以及许多其他事情。相信我——这是一个很长的列表。

你可能已经知道了。

它使用一种配置语言，HashiCorp 配置语言( [HCL](https://www.terraform.io/docs/language/syntax/configuration.html) )，来声明性地定义您的资源的期望状态。您只需在 HCL 中定义资源，运行 Terraform，然后嘭！你最终会在某个平台上获得云资源，比如 GCP、AWS、VMware、Kubernetes，或者其他无数[支持的服务](https://registry.terraform.io/browse/providers)。

那么 HCL 是如何成为某处云中的存储桶的呢？

当您调用 Terraform 来部署您想要的资源时，它会读取您的充满资源定义的 HCL 文件，并从这些定义中推断出一个*计划*。它创建了一个[资源图](https://www.terraform.io/docs/internals/graph.html)——更具体地说，是一个[有向无环图](https://en.wikipedia.org/wiki/Directed_acyclic_graph)——表示资源之间的依赖关系。创建平面图时，Terraform 会遍历此图。该计划直接通知 Terraform 在创建资源时使用的操作顺序，这被称为*应用*。

Terraform 使用一个插件生态系统来管理它和它所管理的基础设施之间的抽象——这些插件被称为*提供者。*提供者包含在给定平台上创建、读取、更新和删除资源所需的所有逻辑。

# 真实世界的例子

让我们通过一个简单的思维练习来演示这个概念。假设您的需求非常简单:您需要部署和更新一个在您的环境中至关重要的 Google Cloud 功能。这是 [Google](https://registry.terraform.io/providers/hashicorp/google/latest/docs) 提供商通过[Google _ Cloud Functions _ function](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/cloudfunctions_function)资源实现的，这使得人们能够部署任意的云功能。

让我们花一点时间来记住 Terraform 将如何部署您的代码，因为即使您像任何其他云资源一样与云功能交互，*我们处理的是代码，这是代码部署。*

Terraform 读取您的资源定义，加载 Google provider，并将资源定义传递给 provide 中的创建、读取、更新或删除(CRUD)函数。如果您感兴趣，请查看 GitHub 上的[代码中的`google_cloudfunctions_function`资源中的当前 CRUD 操作映射。](https://github.com/hashicorp/terraform-provider-google/blob/master/google/resource_cloudfunctions_function.go#L107-L110)

扫描通过云功能[更新](https://github.com/hashicorp/terraform-provider-google/blob/552c3caa141d047303fa03e5d36794280b7d02c1/google/resource_cloudfunctions_function.go#L566)功能代码。这并不复杂，也不神奇。冒着过于简化的风险，它是围绕[函数 API 补丁调用](https://cloud.google.com/functions/docs/reference/rest/v1/projects.locations.functions/patch)的一个薄薄的包装。结果是一种二元状态:云函数更新或出错。如果成功，Terraform 继续前进。如果失败，Terraform 将中止。

就像任何其他云资源一样，Terraform 不知道您的代码。它更新云资源。它就知道这么多。它不知道部署后错误率的增加或请求延迟的增加。它不知道这种新的代码部署是否巧妙地导致了应用程序失败。它不在乎。它只知道在创建或更新资源时可以成功也可以失败。

如果控制平面问题导致无法将您的代码部署到云，它没有回滚的概念，甚至没有前滚的概念。它是快速失效的，需要你立即手动干预。

这种行为不仅限于[云功能](https://cloud.google.com/functions/docs/concepts/overview):从[托管实例组](https://cloud.google.com/compute/docs/instance-groups)到[云运行](https://cloud.google.com/run)容器到 [AppEngine](https://cloud.google.com/appengine) 应用，Terraform 只看到一个云资源。通过/失败。快速失败并中止。

# 好吧，那有什么问题？

首先，也许是最重要的，你完全受供应商逻辑的支配。正如我指出的，大多数提供者的资源级 CRUD 函数都是 API 操作的瘦包装。如果您需要包装在 API 调用周围的更高级的逻辑和控制，这可能意味着您需要在提供者中编写代码。为了用我们的云函数示例来解释这个场景，让我们假设您(开发人员)想在更新操作完成后等待 60 秒，在将操作标记为成功或失败之前检查调用错误率是否升高。简单地说，如果不借助其他手段，如果不修改现有的提供者或编写一个全新的提供者，这是无法用 core Terraform 实现的。

由于 CRUD 函数包装了 API 操作，也许你明白我接下来要说的:你完全依赖于资源控制平面的行为。**对于托管实例组(MIG)这样的资源来说，这是一件好事**，在这里您继承了托管的[更新](https://cloud.google.com/compute/docs/instance-groups/updating-migs)模型和控制平面来管理端到端的流程。对于简单的用例来说，这通常就足够了，但是对于更高级的需求来说，这往往会变得很复杂。记住:Terraform 只理解成功或错误状态，错误状态导致 Terraform 停止。在部署失败的情况下，您是否需要执行回滚步骤、清理或类似的操作？这不是 Terraform 可以自然处理的。

虽然部署流程通常是一系列精确的步骤和检查，但 Terraform 在控制流程方面的投入有限。正如我前面提到的，它在其配置语言 HCL 中使用声明性模型，这导致应用程序根据您想要的状态来推断它的执行顺序。有很多方法可以解决这个问题，但我发现最终的结果往往是一个复杂的、有时难以管理的 web，其中包含了对助手脚本的[引用](https://www.terraform.io/docs/language/expressions/references.html)和无尽的[null _ resource](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)[local-exec](https://www.terraform.io/docs/language/resources/provisioners/local-exec.html)。这使您的资源定义变得复杂，从而更难理解 Terraform 在运行时将执行什么操作。这也在等式中引入了一些可变性:Terraform 直到运行时才能在其计划中考虑 local-exec 的结果，这使得*计划*的输出价值降低！

这里有一个个人的抱怨:曾经因为某种原因不得不去修复一个 Terraform 问题，比如恢复到一个旧的状态文件，手动释放一个状态锁，手动导入资源，或者类似的事情？您能想象在修复失败的应用程序部署*的*之上还要处理那个*吗？*

现在，我知道你在想什么:我非常了解 Terraform，足以解决这些问题。我选择的供应商足够好。控制平面做我需要它做的事情。我的应用没那么重要。

也许这是真的。

# 又有什么意义？

为了在工作中使用正确的工具，你需要承认可用工具的优势*和*局限性。虽然 Terraform 管理资源的简单模型非常适合基础设施，但如果不借助复杂的工具、包装器、插件、框架、模板和其他策略，Terraform 对于编排复杂的部署来说就太有限了，对于软件部署这样重要的事情来说，太笨拙和脆弱了。

这只是我的拙见，在很大程度上，这是持续部署和交付软件的工作。也许这是 Kubernetes 的工作。我的意思是，it *确实*擅长运行和协调任意工作负载的部署和更新。也许是詹金斯管道。也许是运行在[云构建](https://cloud.google.com/build)的管道。也许是三角帆。或者泰克顿。或者通量。这篇文章的重点不是支持任何一个特定的工具，而是强调 Terraform 的不足之处。选择或者写下一个对你有用的工具。

那么这有什么意义呢？我猜是这样的:也许 Terraform 并不总是部署应用程序的错误工具，但它几乎*永远*不是正确的工具。