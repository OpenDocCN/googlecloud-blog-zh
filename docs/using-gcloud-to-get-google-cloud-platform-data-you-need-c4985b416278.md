# 使用 gcloud 获取你需要的 Google 云平台数据

> 原文：<https://medium.com/google-cloud/using-gcloud-to-get-google-cloud-platform-data-you-need-c4985b416278?source=collection_archive---------0----------------------->

![](img/b3a72537f1dc2db8e7347acecdfb42c7.png)

随着你在谷歌云平台上从简单用例转向高级用例，随着你的环境变得更大更复杂，你需要超越与谷歌云服务交互的基本方式。gcloud 是 Google Cloud SDK 提供的一个强大工具，可以让你快速启动并运行。通过一些真实的例子，我将展示如何使用 gcloud 和脚本从 Google 云平台中提取各种数据项，以实现自动化和报告目的。

## 收集项目数据

项目是谷歌云平台最基本的功能之一。项目用于在各种团队、应用程序、环境等之间分配云资源。您可以为项目分配自定义标签，这对报告、跟踪和资源管理有很大帮助。对于你的所有 GCP 资源，包括项目，你一定要有一个强有力的标签策略和标签标准。

## 示例 1:收集项目标签

现在，假设您的环境中有许多项目，您想要提取它们的标签数据。您可能需要它来进行报告(标准遵循报告、成本分配报告等等)。您将使用诸如所有者、业务单元、计费 ID 等标签)和自动化(在资源子集上执行某些自动化活动。您可以使用诸如环境、应用名称等标签)

根据您的数据处理和报告要求，您将只需要提取特定的数据点，并将它们转换成适合您的设置的格式。我在这里考虑的是 CSV 格式，这种格式很容易存储到 BigQuery、ElasticSearch 等存储选项中，或者可以在电子表格的帮助下进行检查。

现在我们来看看 gcloud 提供的关于项目的信息。首先检查 JSON 格式输出总是好的，因为这将清楚地显示各种字段名称和结构。以下是`gcloud projects list` 命令的输出示例:

假设您想要 CSV 格式的项目 ID、项目名称和标签，而不需要诸如创建时间、项目编号等字段。查看输出，您应该确定您想要的字段以及它们的结构。考虑到我们的需求，这里有两个示例命令:

在选项 1 中，您获得逗号分隔的标题行，然后从输出中获得相应的值。因为 labels 是一个嵌套字段(从上面的 JSON 输出可以明显看出)，所以您可以在 CSV 的一个字段中获得所有的标签值，用分号分隔。根据您的数据处理管道，这可能合适，也可能不合适。因此，您可能想要选择选项 2 中所示的特定标签。

另外，您必须小心使用 JSON 输出中正确的字段名。例如，如果您使用`projectid` 而不是`projectId`，您将不会得到预期的输出。

## 收集实例数据

实例支持对报告、跟踪和自动化有用的标签和元数据。我们看到了标签是如何在项目环境中使用的。这同样适用于实例的情况。您可以使用特定于应用程序层、技术层等实例的标签进行细化。实例元数据在管理实例的整个自动化解决方案中扮演着至关重要的角色。例如，拥有自定义元数据(如服务器角色或服务器类型)对您的配置管理工具很有用。环境、补丁窗口等自定义元数据可以帮助您编写有效的自动化脚本。

让我们看看平台为实例提供了哪些数据:

同样，检查 JSON 格式输出有助于您清楚地理解字段和结构。

## 示例 2:收集实例标签和元数据

假设我们想要捕获特定的标签值和元数据值用于报告。下面是一个命令和输出示例

这里不明显的是，实例列表仅来自当前的*项目。如果您想查看另一个项目的实例细节，您需要使用`--project` 标志。但是在现实生活环境中，你会有几十或几百个项目，因此每次都使用`--project`标志不是很有效。您可以使用一个简单的 shell 脚本首先获取所有项目的列表，然后遍历列表以获取每个项目的实例细节。它看起来是这样的:*

示例输出如下所示。如您所见，您可以从所有项目(在本例中，三个不同的项目)中获得实例的详细信息，而不仅仅是一个。

## 示例 3:收集实例操作系统许可证数据

实例数据的另一个有趣的用例是获取所使用的操作系统映像。这将直接影响你每月的账单。您可能想知道有多少实例使用免费和付费操作系统映像。下面的脚本使用`disks.licenses` 字段获取数据:

示例输出如下所示。正如你所看到的，有一个 rhel-7 图像，其余的是 CentOS

## 示例 4:查找孤立磁盘

另一个重要的用例是找出您有多少孤立磁盘，即没有连接到任何服务器的磁盘。如果你不删除你不再使用的磁盘，你将继续支付这些费用。在大环境中，成本会迅速增加。下面的脚本使用`gcloud compute disks list`命令和`users`字段获取详细信息。

示例输出如下所示。

如您所见，磁盘`orphan-disk-1`和`orphan-disk-2`没有任何用户。这意味着它们没有连接到任何服务器。

## 参考资料和进一步阅读

这些只是你可以通过脚本使用 gcloud 从 Google 云平台获取所需数据的许多可能方式中的几种。我希望这些例子能为你提供一个高层次的想法。有关更多详细信息，请参考以下链接。

请告诉我您的想法，以及您如何在您的环境中使用 gcloud。我期待着写另一篇关于使用谷歌云客户端库而不是/另外使用 gcloud 的文章，以及这有什么不同。(编辑:查看“[使用 gcloud 和 Python 客户端库与谷歌计算引擎](/google-cloud/using-gcloud-and-python-client-library-with-google-compute-engine-eaf1b19d8099)”)

*   [https://cloud platform . Google blog . com/2016/06/filtering-and-formatting-fun-with . html](https://cloudplatform.googleblog.com/2016/06/filtering-and-formatting-fun-with.html)
*   [https://cloud.google.com/sdk/docs/scripting-gcloud](https://cloud.google.com/sdk/docs/scripting-gcloud)
*   [https://cloud.google.com/sdk/gcloud/reference/topic/formats](https://cloud.google.com/sdk/gcloud/reference/topic/formats)
*   [https://cloud . Google . com/resource-manager/docs/creating-managing-labels](https://cloud.google.com/resource-manager/docs/creating-managing-labels)

(声明:此处表达的观点为本人观点。信息按“原样”提供。)