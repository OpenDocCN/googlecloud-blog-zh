# 用谷歌顶点人工智能服务机器学习模型

> 原文：<https://medium.com/google-cloud/serving-machine-learning-models-with-google-vertex-ai-5d9644ededa3?source=collection_archive---------0----------------------->

## 以任何规模部署和服务任何类型的机器学习模型。

公司经常将他们的模型部署到虚拟机上(谷歌计算引擎或者甚至是本地机器)。这是应该避免的。谷歌云提供了一个名为 Vertex AI Endpoints 的专用服务来部署你的模型。

与易用性相比，Vertex AI Endpoint 提供了很大的灵活性。您可以保持简单，也可以使用自定义功能根据您的需求进行定制…