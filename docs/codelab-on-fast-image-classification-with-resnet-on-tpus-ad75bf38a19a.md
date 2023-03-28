# TPUs 上基于 ResNet 的快速图像分类 Codelab

> 原文：<https://medium.com/google-cloud/codelab-on-fast-image-classification-with-resnet-on-tpus-ad75bf38a19a?source=collection_archive---------3----------------------->

跟随这个 [codelab，学习如何在张量处理单元上根据自己的数据训练 TensorFlow ResNet 图像分类模型](https://codelabs.developers.google.com/codelabs/tpu-resnet)，并将其部署为微服务。这是从零开始训练最先进的图像分类模型的最快和最便宜的方法之一。

https://codelabs.developers.google.com/codelabs/tpu-resnet

![](img/eec063ed5fc09deb14a854e34a59309d.png)

这看起来不像玫瑰。这是什么？

最棒的是:

*   这些是不需要写的[张量流](https://www.tensorflow.org/)代码(我们[会替你处理好](https://github.com/tensorflow/tpu/tree/master/models/official/resnet))
*   无需安装软件或启动基础设施([云 ML 引擎](https://cloud.google.com/ml-engine/docs/tensorflow/technical-overview)无服务器)
*   你可以在云上训练，但是可以在任何地方部署模型(使用 [Kubeflow](https://github.com/kubeflow/kubeflow)
*   完整的代码是开源的(不是黑盒)。

[笔记本在 GitHub](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/quests/tpu/flowers_resnet.ipynb) 中，这篇更详细的博客文章解释了这个过程。