# 使用云事件和云事件生成器

> 原文：<https://medium.com/google-cloud/using-cloud-events-and-cloud-events-generator-4b71b8a90277?source=collection_archive---------2----------------------->

本文讨论了 CloudEvents 和 CloudEvents Generator 的用法，这有助于您更好地理解本教程系列的其他教程中使用的演示项目。这是[构建事件驱动的云应用和服务](/google-cloud/building-event-driven-cloud-applications-and-services-ad0b5b970036)教程系列的一部分。

# 云事件

CloudEvents 是由[云本地计算基金会](https://www.cncf.io/)的无服务器工作组发起的，目标是标准化事件发布者如何描述他们的事件。目前，计划仍在进行中，规范还没有稳定下来；许多云服务提供商和开源项目已经宣布了他们采用该规范的计划，包括 [Knative Eventing](https://knative.dev/docs/eventing/) 和 [Azure](https://azure.microsoft.com/en-us/blog/announcing-first-class-support-for-cloudevents-on-azure/) 。

> ***注***
> 
> [*你可以在这里*](https://github.com/cloudevents/spec/blob/v0.3/spec.md) *阅读最新发布的规范(0.3)。*

CloudEvent 由许多属性组成，比如事件的 ID 和事件的类型。CloudEvent 规范定义了 CloudEvent 可能具有的必需和可选属性的集合，如下所示:

此外，CloudEvents 允许开发人员通过扩展添加他们自己的属性集。[此处提供了记录的扩展列表](https://github.com/cloudevents/spec/blob/master/documented-extensions.md)。

CloudEvents 绑定有助于跨应用、服务和设备传输事件。例如，您可以将事件绑定到 JSON 格式，或者将其映射到 HTTP 请求。

# 云事件生成器

通常，要构建 CloudEvent，您必须直接使用您喜欢的编程语言的内存结构，或者使用 CloudEvents SDK(也是一项正在进行的工作):

在本系列教程中，为了简单起见，我们使用一个实验项目， [CloudEvents Generator](https://github.com/michaelawyu/cloud-events-generator) ，用于尽可能发布和接收事件。该工具以 JSON 或 YAML 格式的事件模式作为输入，并准备一个您自己的事件库，您可以用它来发布和接收事件。模式输入和事件库也可以帮助团队在事件驱动的系统上更好地协作。例如，要使用 CloudEvents Generator 发送上面代码片段中的事件，首先要指定一个模式，如下所示:

将模式传递给 CloudEvents Generator，并要求它准备一个 Python 包。然后，您可以使用这个包来创建相同的事件:

要查看实际运行的 CloudEvents Generator，请单击下面的按钮在 Cloud Shell 中进行尝试:

[![](img/47b93403044c76424017e0f445096836.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/michaelawyu/cloudevents-generator&tutorial=./examples/basic/tutorial.md&open_in_editor=./README.md)

点击按钮后，应会打开一个交互式教程；如果没有，运行 teachme。/examples/basic/tutorial.md。

如果您喜欢在本地运行它，请参见下面的步骤:

*   [建立你的 Python 开发环境](https://cloud.google.com/python/setup)。安装 Python 3。
*   克隆 CloudEvents Generator Github 库:

```
git clone [https://github.com/michaelawyu/cloudevents-generator](https://github.com/michaelawyu/cloud-events-generator)
```

*   按照`[examples/basic/tutorial.md](https://github.com/michaelawyu/cloud-events-generator/blob/master/examples/basic/tutorial.md)`继续。

# 下一步是什么

了解

*   [反应式事件驱动系统](/@ratrosy/reactive-event-driven-systems-and-recommended-practices-785a1ab7e509)
*   [流处理事件驱动系统](/@ratrosy/introduction-to-event-driven-systems-with-stream-processing-8b169a9fae12)