# 用顶点 AI 训练来训练机器学习模型

> 原文：<https://medium.com/google-cloud/how-to-train-ml-models-with-vertex-ai-training-f9046bfbcfab?source=collection_archive---------0----------------------->

## 使用自定义容器的简单且可扩展的方法

我定期指导客户如何扩展他们在云上的培训。虽然每个客户都有独特的要求和期望，但总体流程是相同的。

在本文中，我们用顶点 AI 训练一个变压器模型*(*[*)DistilBert*](https://arxiv.org/abs/1910.01108)*)*。作为机器学习框架，我用的是 TensorFlow。它也将适用于任何其他机器学习框架(PyTorch，XGBoost，scikit-learn，*你能想到的*)和任何其他 ML 使用…