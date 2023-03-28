# 用 Terraform 触发数据流？以下是如何使用 Flex 模板将任何数据流运行器选项传递到您的工作中

> 原文：<https://medium.com/google-cloud/using-terraform-to-trigger-dataflow-2698a4814196?source=collection_archive---------3----------------------->

![](img/7940d1463dc603293af79644c1ea61a6.png)

照片由 [SpaceX](https://unsplash.com/@spacex?utm_source=medium&utm_medium=referral) 在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 上拍摄

在许多可以用来触发数据流作业的不同方法中，Terraform 比较受欢迎。Terraform 使用[数据流 API](https://cloud.google.com/dataflow/docs/reference/rest) 来触发数据流作业。这意味着您只能启动模板。但是由于有了 [Dataflow Flex 模板](https://cloud.google.com/dataflow/docs/guides/templates/using-flex-templates)，这实际上并没有对你可以启动的工作种类施加限制。如果你的代码不是一个模板，就用一个启动容器包装它，使用 Flex 模板。

然而，并非所有的情况都是乐观的。如果您将部署数据流管道的[选项](https://cloud.google.com/dataflow/docs/guides/specifying-exec-params)与 Terraform 资源提供的选项进行比较(常规模板的[选项](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/dataflow_job)，flex 模板的[选项](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/dataflow_flex_template_job)，您会注意到数据流运行器可以接受的选项与您使用 Terraform 传递的选项之间存在差距。

我们如何弥合这一差距？

有几种方法可以将选项传递给数据流管道。命令行不是唯一的方法。您可以始终以编程方式设置流道选项。为此，您可以使用类`[DataflowPipelineOptions](https://beam.apache.org/releases/javadoc/2.27.0/org/apache/beam/runners/dataflow/options/DataflowPipelineOptions.html)`(在 Java 中)或`[GooogleCloudOptions](https://beam.apache.org/releases/pydoc/2.27.0/_modules/apache_beam/options/pipeline_options.html#GoogleCloudOptions)`(在 Python 中)。

如何使用这些类来设置数据流选项？问题是，这些数据流选项中有许多没有使用 Terraform 资源公开。但是您可以使用`DataflowPipelineOptions`或`GoogleCloudOptions`从代码中设置它们。

在 Python 中，我们可以使用`view_as`对`GoogleCloudOptions`进行“造型”:

在 Java 中，我们需要做更多的改变。

首先，如果你正在使用由 [Apache Beam Quickstart](https://beam.apache.org/get-started/quickstart-java/) 提供的默认`pom.xml`文件，你将需要确保数据流依赖在编译时可用。确保将其包含在您的`<dependencies>`部分:

```
<dependency>
  <groupId>org.apache.beam</groupId>
  <artifactId>beam-runners-google-cloud-dataflow-java</artifactId>
  <version>${beam.version}</version>
</dependency>
```

一旦你有了这种依赖，你应该能够在你的管道中使用`DataflowPipelineOptions`。您可以将常规的`PipelineOptions`转换为`DataflowPipelineOptions`，并设置数据流参数:

通过这些示例，您可以设置此表中包含的任何数据流运行程序选项:[https://cloud . Google . com/data flow/docs/guides/specifying-exec-params # setting-other-cloud-data flow-pipeline-options](https://cloud.google.com/dataflow/docs/guides/specifying-exec-params#setting-other-cloud-dataflow-pipeline-options)

是的，很好，但是所有的代码片段都有硬编码的数据流选项，您想知道如何在运行时设置这些选项。

如果你写的是常规模板，**就不能**。

使用常规模板，您的管道选项将是`ValueProviders`，您将需要在创建管道之前尝试“获取”它们(调用 *get* 方法)。那会给你一个错误， *get* 只在你的管道已经在运行的时候才起作用。

但是不用担心，你可以用**数据流灵活模板**解决这个问题。

在 Flex 模板中，你可以有“普通的”运行时选项(不是`ValueProvider`)，你可以像“普通的”工作一样编写代码，而不必处理任何与模板相关的事情。一旦你写了代码，你需要用一个*启动器容器*包装它。

数据流文档中有一个如何将你的代码转换成 Flex 模板的例子，以及 Google 提供的关于容器的所有细节:[https://cloud . Google . com/data flow/docs/guides/templates/using-Flex-templates](https://cloud.google.com/dataflow/docs/guides/templates/using-flex-templates)

使用 Flex 模板，您可以公开一个定制选项，比如`--i-want-streaming-engine`，获取它的值，然后将其传递给 Dataflow options 对象。

然后，在 Terraform 中，您将在您的`parameters`部分中指定该选项:

您可以更新您的选项来读取您的自定义选项，然后将它们的值传递给相应的数据流选项。

例如，在 Python 中，我们可以做如下事情:

在 Java 中的对等词是:

数据流变化很快，新的功能和选项以一定的频率出现。并非所有这些选项都可以立即从 Terraform 资源和模块中获得，以便与您的数据流模板一起使用。但是，您总是可以从代码中以编程方式设置这些选项。

如果您使用常规数据流模板，这些运行器选项必须硬编码在您的源代码中。这是因为管道选项只能通过*值提供者*获得，在运行时无法恢复，只有在管道已经启动之后。

但是如果你使用 Dataflow Flex 模板，这不是问题。只需在管道中添加一个普通的定制选项，并编写一些代码将这些值传递给数据流选项对象。然后只需在您的`google_dataflow_flex_template_job`的`parameters`部分设置您的自定义选项，您将能够使用任何数据流选项，无论它们是否可通过 Terraform 获得。