# GCP 的自然语言自动文档分类魔法

> 原文：<https://medium.com/google-cloud/gcps-natural-language-automl-magic-for-document-classification-79e0188afc1a?source=collection_archive---------1----------------------->

![](img/0370031a1dc45a952cf7481c0a068da5.png)

图片来源:[https://globalcloudplatforms . com/2020/11/07/introducing-document-ai-platform-a-unified-console-for-document-processing/](https://globalcloudplatforms.com/2020/11/07/introducing-document-ai-platform-a-unified-console-for-document-processing/)

文本和文档分类是行业中非常普遍的 ML 用例，其中大量的文本信息是每个部门的驱动源。无论是零售、医疗保健、电子商务、汽车、银行还是金融，总会有需要文本和文档分类的用例。输入可以是图像、文本段落、HTML 页面或 PDF 格式。

PDF 和图像是最常用的行业文档格式。对文本片段进行分类比对 PDF 或图像文档进行分类更容易。对 pdf 或文本图像进行分类面临着从 PDF 或图像中提取文本的挑战。为了实现这一点，开发人员可能的解决方案是在分类之前使用 OCR 引擎提取文本。但是将 OCR 引擎添加到管道中会产生额外的开销，影响解决方案的开发和响应时间。此外，不要忽视分类的性能将依赖于 OCR 提取的性能。此外，很多时候，在文本提取之后，文本的原始格式没有被保留，这可能导致分类信息的丢失，如果文档的结构是您的分类用例中的一个基本特征的话。

这就是 Google 的自然语言文本和文档分类模型成为神奇解决方案的地方。谷歌的自然语言文本和文档分类 AutoML 模型使机器学习专业知识有限的开发人员能够训练和交付针对客户业务需求的高质量模型，并在几分钟内构建自己的定制机器学习模型。Google Cloud 的 AutoML 解决方案允许我们直接使用 PDF 文件作为 AutoML 模型的分类输入，无需在分类前执行任何文本提取。我使用 AutoML 作为我的用例，其中数据集是高度非结构化的，并且在一个类中存在许多变化。然而，AutoML 模型的性能在训练期间为 96%,对于未训练的测试样本超过 90%。我建议使用 AutoML 进行文本和文档分类，尤其是在处理 PDF 或图像文档时。

为了训练您的自定义模型，您需要提供要分析的文档类型的代表性样本。开发人员需要记住，训练数据的质量会极大地影响您创建的模型的有效性，进而影响从该模型返回的预测的质量。因此，尽量为您的用例包含尽可能多的变化。

## 如何使用 GCP AutoML 对文本文档进行分类

1.  登录您的 GCP 帐户
2.  在 GCP 产品的**左侧面板**中，找到自然语言。
3.  转到**自然语言 API**
4.  选择**自动文本和文件分类**
5.  选择**顶部的新数据集**按钮
6.  添加数据集名称，选择区域，如果是每个文档仅需要一个标注的用例，则选择单标注分类，如果文档存在多个标注，则选择多标注分类，然后添加要创建数据集的区域，然后单击创建数据集。关于数据集创建的详细描述可以在[https://cloud . Google . com/natural-language/automl/docs/datasets](https://cloud.google.com/natural-language/automl/docs/datasets)找到。要通过代码创建数据集，请遵循[https://cloud . Google . com/natural-language/automl/docs/datasets # python](https://cloud.google.com/natural-language/automl/docs/datasets#python)
7.  一旦数据集被创建，您将看到它被添加到**数据集页面**。
8.  点击您创建的数据集，并转到**导入选项卡**。
9.  有三种方法可以导入数据集:I)从本地上传 CSV(*CSV 应包含文件的 GCS URI，以类名*分隔)，ii)从 Google 云存储(GCS)上传 CSV(*CSV 应包含文件的 GCS URI，以类名*分隔)。 iii)通过直接上传一个 zip 文件夹(*zip 文件夹必须包含带有类别名称的子文件夹，并且在每个子文件夹中，该类别的文档必须存在于本地*。 关注[https://cloud . Google . com/natural-language/automl/docs/datasets](https://cloud.google.com/natural-language/automl/docs/datasets)了解更多关于数据集导入的信息。要通过代码导入数据，请关注[https://cloud . Google . com/natural-language/automl/docs/datasets # python](https://cloud.google.com/natural-language/automl/docs/datasets#python)
10.  在数据导入期间，您还需要在 GCS 中提供一个目标存储桶，AutoML 在导入数据集时将该存储桶用作输出存储桶。
11.  单击导入。数据集的导入通常需要时间。数据导入后，您的 GCP 帐户将收到一封电子邮件通知。
12.  导入结束后，您将能够在您创建的数据集内的**项目选项卡**中看到您的带有指定类别标签的文件。
13.  一旦标签可用，点击**训练标签**并开始训练。培训需要时间，一旦培训完成，您将收到一封电子邮件通知。所需的训练时间取决于几个因素，如数据集的大小、训练项目的性质和模型的复杂性。通过 python 代码训练你的模型，关注[https://cloud . Google . com/natural-language/automl/docs/models # python](https://cloud.google.com/natural-language/automl/docs/models#python)。跟随文档[https://cloud . Google . com/natural-language/automl/docs/models](https://cloud.google.com/natural-language/automl/docs/models)对 AutoML 模型训练有更多了解。
14.  在模型被训练之后，您进入**测试并使用标签**并部署您的模型。模型部署需要几分钟时间，在部署完成后，您会收到一封电子邮件通知。
15.  一旦模型得到部署，您可以使用 GCP AutoML GUI 测试模型，方法是从 GCS 中选择一个文件，然后单击**测试和使用选项卡**中的预测按钮。有关预测的更多详情，请关注[https://cloud . Google . com/natural-language/automl/docs/predict](https://cloud.google.com/natural-language/automl/docs/predict)。为了通过脚本获得模型的预测，您可以使用 Google 的 AutoML 客户端库，并在代码中调用模型的预测端点。响应是 JSON 格式的。使用 python 客户端库的文档[https://cloud . Google . com/natural-language/automl/docs/predict # python](https://cloud.google.com/natural-language/automl/docs/predict#python)。AutoML 还为您提供了执行批量预测的灵活性。如果您想进行批量预测，请关注[https://cloud . Google . com/natural-language/automl/docs/predict # python](https://cloud.google.com/natural-language/automl/docs/predict#python)
16.  要检查模型的性能矩阵，请转到**评估选项卡**。AutoML 自然语言还提供了一组总的评估指标，表明模型的整体表现如何，以及每个类别标签的评估指标，表明模型对该标签的表现如何。您将从训练期间执行的训练测试分割的测试数据中找到模型的性能。AutoML 对输入数据集执行 80–10–10 的自动训练、测试和验证拆分。您还可以编辑模型的**置信度阈值**，以根据您的用例调整精度和召回值。关于 AutoML 使用的性能矩阵的解释可以在[https://cloud . Google . com/natural-language/AutoML/docs/evaluate](https://cloud.google.com/natural-language/automl/docs/evaluate)找到。要通过代码获得评估矩阵，请遵循[https://cloud . Google . com/natural-language/automl/docs/evaluate # python](https://cloud.google.com/natural-language/automl/docs/evaluate#python)

在我的下一篇博客中，我将解释 AutoML 上的同步和异步调用，以及如何管理 AutoML 的长时间运行的操作。

# 快乐学习。继续跟着！！！