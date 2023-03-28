# 谷歌云平台的 a 到 Z 个人选择— D 是部署经理

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-d-is-for-deployment-manager-f40b00b53276?source=collection_archive---------0----------------------->

部署管理器允许您创建一组声明性模板，这些模板允许您一致地部署[云平台资源](https://cloud.google.com/deployment-manager/configuration/#listing_available_resource_types)。

这意味着您可以在不同的项目和环境中设置相同的配置。例如，准确反映生产的测试环境。您也不需要让副本环境一直保持运行，因为您可以放心地使用 Deployment manager 轻松地再现一个环境。

部署管理器通常使用两种文件

一个[配置文件](https://cloud.google.com/deployment-manager/configuration/)，它是一个 YAML 文件

[模板文件](https://cloud.google.com/deployment-manager/configuration/adding-templates)分别是 [Jinja 2.7.3](http://jinja.pocoo.org/docs/dev/) 或 [Python 2.7](https://www.python.org/download/releases/2.7/)

还有一个[模式文件](https://cloud.google.com/deployment-manager/configuration/using-schemas)，但我不会在这篇文章中谈论它，你可以跳到文档中去看看。

在这篇文章中，我将只关注配置和模板文件，希望能让您理解为什么在使用部署管理器时通常会有这两种类型的文件。

至少需要一个[配置文件](https://cloud.google.com/deployment-manager/configuration/create-configuration-file#configuration_structure)才能传递给部署管理器。该文件定义了您的部署的外观。它定义了您希望部署的资源和设置，如区域、机器类型等。

要获得受支持资源和相关属性的列表，您可以使用此处的[表](https://cloud.google.com/deployment-manager/configuration/create-configuration-file#supported_resource_types_and_properties)或使用 gcloud 命令从命令行检索列表:

```
gcloud deployment-manager types list
```

要部署一个配置，您需要通过 [gcloud 命令](https://cloud.google.com/deployment-manager/configuration/create-configuration-file#provide_a_configuration_through_gcloud)或 [API](https://cloud.google.com/deployment-manager/configuration/create-configuration-file#provide_a_configuration_through_the_api) 将配置文件传递给部署管理器。

现在，一个巨大的配置文件可能会变得难以管理，因此模板就派上了用场。模板是一种将配置分解成逻辑可组合单元的方法，您可以单独更新和重用这些单元。

要将这些模板作为您的配置的一部分，您可以通过使用调用模板的配置文件中的 [imports 属性](https://cloud.google.com/deployment-manager/configuration/adding-templates#import_the_template)来导入它们(从这里开始，我将引用将模板作为主配置文件导入的配置文件)。

模板在许多方面都非常灵活:

可组合性——意味着更容易管理和维护定义。它还使得在不同的部署中重用定义变得容易。您可能不想重新创建配置文件中包含的所有内容，但可能只想在定义要使用的实例时保持一致性。

[模板变量](https://cloud.google.com/deployment-manager/configuration/adding-templates#template_variables) -是重用模板的一种简单方法，因为它允许你通过在配置文件中声明要传递给模板的值来抽象属性。这意味着您可以为每个配置更改它的值，而不必更新模板。例如，您可能希望将不同区域中的测试实例部署到生产实例中，因此您在模板中声明了一个变量，该变量继承了主配置文件中的区域值。

[环境变量](https://cloud.google.com/deployment-manager/configuration/adding-templates#environment_variables) —您声明的变量允许您在不同的项目和部署中轻松地重用模板。因此，这些是项目 ID 或部署名称之类的东西，而不是您想要部署的资源的属性，这就是模板变量的用途。

为了区分这两种类型的变量，让我们假设您有两个希望部署实例的项目，但是您知道您希望将这些实例部署到每个项目的不同区域中。您可以为项目 ID 和部署名称声明环境变量，并为要部署的实例的区域资源属性使用模板变量。

模板模块——非常酷，因为它们可以被其他模板用作模块。你想什么时候做这个？假设您想要部署同一个部署的阶段、测试和生产版本，并且理想情况下它们是相同的配置，但是您想要一种方法来通过分配给实例的名称来标识部署的每个实例。最简单的方法是将每个资源命名为应用程序名称—环境，例如阶段、测试或生产。因此，与其声明大量的模板变量，不如使用一个助手模板，它将根据环境自动生成实例的名称(现在这变成了一个过载的术语——对不起！) .docs 有一个使用模板模块的好例子。

如果你已经做到了这一步，你可能想知道为什么 Jinja 和 python。这真的是一个选择的问题。Jinja 很容易映射到 YAML，所以如果你只是想要可组合性和所有给你的东西，开始使用 Jinja 和用 YAML 写的配置文件并不是一个巨大的飞跃。

使用 python 使您能够以编程方式创建模板文件。您可以混合搭配 jinja 和 python 模板，因此，如果您从 jinja 文件开始，然后决定需要以编程方式定义一些模板，并希望使用相同的主配置文件，您可以这样做。

是的，我知道任何在 twitter 上关注我的人可能都知道我对任何让你关心缩进和空白的事情的不妥协，所以通过推理，我真的不是 YAML 的粉丝，但除此之外，Deployment manager 确实将基础设施作为代码来处理，我非常相信这一点，所以这推翻了我的反对意见。虽然我仍然不喜欢 YAML，但我喜欢部署管理器！