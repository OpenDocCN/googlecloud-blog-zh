# 如何将谷歌云平台与自己喜欢的机器学习框架结合使用

> 原文：<https://medium.com/google-cloud/how-to-use-google-cloud-platform-with-your-favorite-machine-learning-framework-ca59f1e81aee?source=collection_archive---------2----------------------->

还记得你的算法需要几个小时才能完成的时候吗？如果你需要运行它 10 (100？)倍！在这种情况下，可能值得考虑扩大规模，谷歌云平台有很多选项可以让你运行你最喜欢的机器学习框架/算法。

在一个典型的 ML 管道中，有(至少)三个关键步骤。

1.  你**训练**一个模型来优化你的目标函数(某种损失/误差函数)。
2.  然后，您可能想要探索不同的模型配置(例如，模型的类型、层数)，看看哪一个效果最好:这是**超参数调整**部分。
3.  最后，一旦您找到了最佳配置并对结果感到满意，您就想公开您的模型以使其可用于运行预测。**将您的模型**部署为 API 是将其公开给生产应用程序的好方法，因为该接口为开发人员所熟知并普遍使用。

在本帖中，我们将回顾这些步骤，并将重点放在一些常见的框架上:

*   **TensorFlow** : Google 开源库，多用于深度学习应用，扩展性高。
*   **Scikit-learn** :机器学习库，当数据适合内存时，主要用于非神经网络应用。
*   **XGBoost** :实现一个模型(极端增强渐变)的库，流行于具有高可伸缩性的结构化数据。
*   **H2O** :机器学习库，多用于非神经网络应用，扩展性高。

**这篇文章旨在给出一些最佳实践的指南，并不作为解释每个过程的每一步的教程。**但是，相关教程的链接会在可用时提供。

Cloud ML Engine 是运行 TensorFlow 库的一个很棒的工具，在产品[网站](https://cloud.google.com/ml-engine/docs/tensorflow/technical-overview)上有很好的文档记录。因此，这篇文章将关注其他三个库:Scikit-learn、XGBoost 和 H2O。

# 培养

在云上训练模型提供了两个主要优势:水平扩展和访问更高端的加速器，如 GPU 和 TPU。

## GCE 上的缩放

谷歌计算引擎(GCEs)或托管 Jupyter 笔记本电脑(如 Kaggle 内核或 Datalab)上的训练模型是扩展的简单方法。简单地旋转一个实例，然后像在本地一样训练一个模型！

这种方法的优点是:

*   **垂直缩放**通过使用任意强大的 GCE 实例(不需要修改代码)。
*   **通过使用启用 GPU 的 GCE 实例进行 GPU 加速**(scikit-learn 无法实现，但 [xgboost](http://xgboost.readthedocs.io/en/latest/gpu/) 和 [h2o](http://docs.h2o.ai/driverless-ai/latest-stable/docs/userguide/install/google-compute.html) 支持)。
*   **在底层库启用时使用多核进行水平扩展***(感兴趣的水平扩展参数:scikit learn 中的一些模型有参数*[*【n _ jobs】*](http://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html%5C)*，Xgboost 有参数*[*【n threads】*](https://machinelearningmastery.com/best-tune-multithreading-support-xgboost-python/)*，它将处理多线程，而 h2o.init()将自动检测内核数量)。*

## 通过 Dataproc 对 Spark 进行缩放

H2O 和 XGBoost 可以在 Spark 上运行，因此可以使用 Dataproc 在多个 workers 上扩展 Spark。Scikit-learn 还有一个 [Spark 版本](https://github.com/databricks/spark-sklearn)，你可以利用它。虽然通过 Dataproc 在 Spark 上进行缩放允许您使用更大的数据集，并提供更好的水平缩放，但它确实需要更多的时间来设置。

# 超参数调谐

在云中执行超参数调优为您提供了水平扩展的优势。有几种潜在的方法可以做到这一点。

如果您使用 GCE 来训练单个模型，您可以继续使用此训练环境来进行 hyper 参数调整，但是您的水平缩放将受限于您的机器的核心数量。

如果您已经通过 Dataproc 使用 Spark 来训练单个模型，一个自然的想法是继续使用相同的环境。所有的 ML 库都包含一些函数，可以进行一些网格搜索探索，为您处理缩放，所以不需要额外的设置。

TPOT 也是一种调整你的模型的便捷方式。它与 scikit-learn 非常相似，可以帮助您对最佳模型进行基准测试。它会搜索模型和参数的最佳组合，从您的解决方案中挤出一两个百分点！通过使用 n_jobs 参数，您可以在一个 GCE 实例上扩展这个搜索。

# 模型部署

在部署方面，云 ML 引擎可以为您节省大量时间。您不必考虑使用正确的组件(例如负载平衡、版本控制)来构建基础设施，只需几步就可以将您训练过的 scikit-learn 和 XGBoost 模型部署为 API([示例](https://cloud.google.com/blog/big-data/2018/04/serving-real-time-scikit-learn-and-xgboost-predictions)，参见下面的更多链接)。

对于 H2O，仍然没有现成的解决方案，因此需要使用 Flask(仅用于开发)/App Engine/Google Cloud 端点来完成一些工作。一些例子可以在[这里](https://www.analyticsvidhya.com/blog/2017/09/machine-learning-models-as-apis-using-flask/)(第六部分)或者[这里](https://github.com/GoogleCloudPlatform/ml-on-gcp/tree/master/sklearn/gae_serve)找到。

# 每个框架的摘要

**云 ML 引擎流量**

*   训练: [Cloud MLE](https://cloud.google.com/ml-engine/docs/tensorflow/training-overview) 通过使用加速器(GPU)和集群(多工)来加速你的模型。
*   超级参数:[云 MLE](https://cloud.google.com/ml-engine/docs/tensorflow/hyperparameter-tuning-overview) 让你用贝叶斯优化调整你的参数。
*   部署:[云 MLE](https://cloud.google.com/ml-engine/docs/tensorflow/prediction-overview) 将您的模型部署为 API。

**Scikit-learn**

*   训练:您可以在 GCE 上训练您的模型并缩放实例。缩放将取决于模型本身，特别是是否有参数 *n_jobs* ( [示例](http://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html%5C))。
*   Hyper 参数:GCE(例如 GridSearch 函数中的 n_jobs)。您还可以通过使用 Spark 库 [spark sklearn](https://databricks.com/blog/2016/02/08/auto-scaling-scikit-learn-with-apache-spark.html) 在 Dataproc 上进行缩放。
*   部署:[云 MLE](https://cloud.google.com/ml-engine/docs/scikit/getting-predictions) 将您的模型部署为 API(示例在[笔记本](https://github.com/GoogleCloudPlatform/cloudml-samples/blob/master/sklearn/notebooks/Online%20Prediction%20with%20scikit-learn.ipynb)中)。

**XGBoost**

*   训练:你可以在 GCE 上训练你的模型，使用参数 *n_threads* 进行缩放。您也可以使用 [Spark 版本](https://docs.databricks.com/user-guide/faq/xgboost.html)并在 Dataproc 上训练您的模型。
*   Hyper parameter:缩放由内置的 xgboost gridsearch 处理。
*   部署:[云 MLE](https://cloud.google.com/blog/big-data/2018/04/serving-real-time-scikit-learn-and-xgboost-predictions) 将您的模型部署为 API(示例在[笔记本](https://github.com/GoogleCloudPlatform/cloudml-samples/blob/master/sklearn/notebooks/Online%20Prediction%20with%20scikit-learn.ipynb)中)。

**H2O**

*   训练:你可以在 [GCE](http://docs.h2o.ai/driverless-ai/latest-stable/docs/userguide/install/google-compute.html) 上训练你的模型，并缩放实例。您也可以使用 [Spark 版本](https://github.com/h2oai/sparkling-water)并在 Dataproc 上训练您的模型。对于 R 用户，你可以在 GCP 上启动 [R-Studio 并使用合适的](https://console.cloud.google.com/launcher/details/rstudio-launcher-public/rstudio-server-pro-for-gcp) [h2o 包](https://www.h2o.ai/h2o-old/h2o-on-r/)。
*   超级参数:缩放由内置的 h2o 网格搜索处理。
*   部署:您可以使用 Flask/App Engine/Cloud Endpoints 等工具来进行部署(一些示例可以在这里的[和这里](https://www.analyticsvidhya.com/blog/2017/09/machine-learning-models-as-apis-using-flask/)的[中找到)](https://github.com/GoogleCloudPlatform/ml-on-gcp/tree/master/sklearn/gae_serve)

**词汇:**

*   [谷歌计算引擎](https://cloud.google.com/compute/) (GCE):谷歌云上的可扩展虚拟机。
*   [云机器学习引擎](https://cloud.google.com/ml-engine/docs/tensorflow/technical-overview) (Cloud MLE):面向机器学习应用的 Google Cloud 托管基础设施。
*   [Dataproc](https://cloud.google.com/dataproc/) :在 Google Cloud 上运行 Apache Spark 或 Hadoop 的托管服务。

我希望这篇文章能帮助你澄清不同的选择！还有，如果你有其他想法，请在评论里分享！

感谢您的阅读！