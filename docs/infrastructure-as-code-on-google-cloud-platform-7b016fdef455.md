# 谷歌云平台上的基础设施代码

> 原文：<https://medium.com/google-cloud/infrastructure-as-code-on-google-cloud-platform-7b016fdef455?source=collection_archive---------2----------------------->

**第一天:发布/订阅**

**更新 2017.01.12:** 我搞清楚了 Pub/Sub 权限和引用。有关更多信息，请参见“更新”部分。

对于那些还没听说的人来说，这几天发生了一整件 [DevOps](https://en.wikipedia.org/wiki/DevOps) 的事情。话又说回来，如果你读过这篇文章的标题，你可能已经知道了。迄今为止，我的大部分 DevOps 旅程都围绕着自动化代码部署和配置管理。本质上，这意味着大量的[木偶](https://puppet.com/)、[厨师](https://www.chef.io/)、[卡皮斯特拉诺](https://github.com/capistrano/capistrano)、[詹金斯](https://jenkins.io/)等等。

我曾经读过关于基础设施即代码(IaC)的文章，但是直到现在我都没有机会实现它。我主要在主要是物理硬件或者主要外包给托管数据中心的环境中工作。但现在我可以先潜入水中了！我们正在将我们的整个应用基础设施迁移到谷歌云平台(GCP)！呜呜呜。

你的第一个问题可能是为什么是 GCP？我知道，我知道。AWS 是目前大规模云部署的事实标准。选择谷歌而非亚马逊既有商业原因，也有技术原因。我并不了解影响最终决定的所有因素(即使我知道也不会告诉你——专有信息等等)。我所知道的是，谷歌的定价更便宜，性能也很出色。

使用 GCP 而不是 AWS 的最大缺点是 GCP 的特性集明显落后于 AWS。可用的功能一直在快速向前发展，谷歌在测试阶段就提供了许多即将推出的功能供测试，但这表明 AWS 已经做了更长时间。此外，虽然 GCP 工具集(我指的是 API 和命令行工具)非常好，但它们不如 AWS 选项成熟。

我们(这里的“我们”指的是我的工作团队)已经将基础设施的一部分转移到 GCP 大约一年了。大部分时间我都在感叹这个过程的手工性质，但是一天就那么几个小时。现在，我们正将更多的应用基础设施转移到 GCP，包括开发、质量保证、试运行和生产环境。是时候 A)对基础设施应该是什么有某种记录，B)自动化这该死的东西，以便我们每次都做对(或者至少以一致和可识别的方式做错)。

输入基础设施代码。本周，我和一位同事开始为 GCP 测试基础设施自动化工具。我正在使用谷歌的`[gcloud deployment-manager](https://cloud.google.com/deployment-manager/deployments/)`命令行工具，我的同事正在使用哈希公司的 [Terraform](https://www.terraform.io/) 。到目前为止，两者都做得很好，但是他们都有一些缺点。我不打算在这里进行比较。我将用 deployment-manager 记录我的旅程，也许有人会发现它有助于克服我曾经面临的一些障碍。或者你可能会说“他还不知道吗？真是个傻逼！”

当我梳理 Google 的文档时，许多例子都围绕着创建新的实例。耶！这才是大多数人需要的！话说回来，我不是大多数人。我将部署的大部分是容器化的，并在 kubernetes 中运行，所以我需要的部分都是关于网络、负载平衡和服务帐户等。

部署管理器使用 YAML 文件来描述所需的配置。我在文档中找到了一个很棒的[页面，描述了为 101 种不同类型创建配置文件和 API 文档的链接。我想做的第一件事是创建一个非常简单的发布/订阅主题和订阅。我点击了 API 文档的链接，然后 404。好吧，谷歌，你没有帮我。进一步的研究表明，种类也远不止 71 种:](https://cloud.google.com/deployment-manager/docs/configuration/syntax-reference)

```
[dbayer@LVML201033: ~]$ gcloud deployment-manager types list | wc -l
     114
```

最终，通过大量的试验、错误和咬牙切齿，我找到了大部分。正如你在代码末尾看到的，我还在纠结最后一项:

```
resources:
- name: dav-pubsub-test-01
  type: pubsub.v1.topic
  properties:
    topic: dav-depmgr-test
- name: dav-pubsub-sub-01
  type: pubsub.v1.subscription
  metadata:
    dependsOn:
    - dav-pubsub-test-01
  properties:
    subscription: dav-depmgr-test-sub
    topic: projects/davs-test-project/topics/dav-depmgr-test
    ackDeadlineSeconds: 600
# TODO: figure out how to apply permissions to topic/subscription
# TODO: figure out how to specify topic by 
#       reference in the subscription
```

尽管设置发布/订阅很简单，选项也很少，但我在这个过程中学到了一些重要的东西:

*   使用 deployment-manager，所有资源都是并行创建的。如果省略 dependsOn 块，订阅将在首次运行时失败，因为主题尚未完成。
*   配置选项的合理猜测是 PITA。

请继续关注下一期，我将演示如何创建服务帐户以及当天引起我兴趣的任何东西。

**更新**

**2017.01.12**

在暂时搁置这个问题之后，我现在又回到了这个问题上，并从原始代码块中找出了两个 TODOs。

对于权限(IAM 策略)分配，您必须首先授予项目所有者的 Google APIs 服务帐户权限。这个服务帐户被列为 project#@cloudservices.gserviceaccount.com(例如:`0123456789012@cloudservices.gserviceaccount.com`)。一旦完成，您就可以将 accessControl 块添加到与`properties`相同层次的资源定义中。

对于通过引用为主题分配订阅，只需要找到要检查的正确对象属性。我通过查看 Ruby gem google-cloud-pubsub 的文档得出了这个结论，然后做了一些猜测。

生成的 yaml 文件如下所示:

```
resources:
- name: dav-pubsub-test-01
  type: pubsub.v1.topic
  properties:
    topic: dav-depmgr-test
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.viewer
        members:
        - "user:myuser@example.com"
        - "serviceAccount:mysvcacct@project.iam.gserviceaccount.com"
      - role: roles/pubsub.publisher
        members:
        - "serviceAccount:mysvcacct@project.iam.gserviceaccount.com"
- name: dav-pubsub-sub-01
  type: pubsub.v1.subscription
  properties:
    subscription: dav-depmgr-test-sub
    topic: $(ref.dav-pubsub-test-01.name)
    ackDeadlineSeconds: 600
  accessControl:
    gcpIamPolicy:
      bindings:
      - role: roles/pubsub.viewer
        members:
        - "user:myuser@example.com"
        - "serviceAccount:mysvcacct@project.iam.gserviceaccount.com"
      - role: roles/pubsub.subscriber
        members:
        - "serviceAccount:mysvcacct@project.iam.gserviceaccount.com"
```

再举一个例子，上面分配的服务帐户权限是您的应用程序成功使用 Google Pub/Sub 的最低要求。您还可以在部署中创建服务帐户，然后在分配权限时使用对该帐户的引用。

您可能会注意到，我从订阅中删除了依赖块。通过使用对主题的引用，它创建了一个隐式的依赖，我不再需要显式地声明它。保留依赖块不会导致任何问题，它只是不再必要。