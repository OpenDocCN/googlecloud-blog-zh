# 在 Apache Airflow 和 Cloud Composer 中支持最新的 Google Cloud operators

> 原文：<https://medium.com/google-cloud/backporting-google-cloud-operators-in-apache-airflow-34b6c9efffc8?source=collection_archive---------1----------------------->

![](img/a16f446fe905103eb9a35cfde33c77d3.png)

劳尔·卡乔·奥斯在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 拍摄的照片

Apache Airflow 的 Google Cloud operators 提供了一种从 DAG 连接到 BigQuery、Dataflow、Dataproc 等服务的便捷方式。如果您没有使用最新的 Airflow 版本，您可能会被一组没有提供最新功能的操作符所困扰。

随着云服务的快速发展，这可能会成为一个问题，阻止您在数据管道中利用最新的谷歌云功能。在这篇文章中，我们展示了如何反向移植最新版本的谷歌云操作器，重点是 [Cloud Composer](https://cloud.google.com/composer) (谷歌云的 Apache Airflow 管理版本)，这样你就可以使用它们，而不必被迫升级你的 Airflow 版本。

对于这篇文章，我们假设你使用的是 Airflow 1.10.6。使用 Cloud Composer 的最新可用版本是 1.10.10 。因此，在本例中，系统中默认可用的 Google Cloud 操作符是 Airflow 1.10.6 中附带的操作符。

例如，最新版本的`DataprocSubmitJobOperator`有一个额外的选项，如果 Airflow web UI 重新启动，这个选项会很方便。您可以重新附加到现有作业或再次启动它(参见 [AIRFLOW-3211](https://issues.apache.org/jira/browse/AIRFLOW-3211) )。

如果您在旧版本的 Airflow 中，并且想要使用该行为，该怎么办？**如何更新 Composer 以将这些操作员引入我当前的环境？**

你不必更新你现有的气流版本。您可以将最新版本的 Google Cloud operators 应用到部署在 Composer 实例中的 Airflow 版本，而无需任何停机时间或集群升级过程。此处显示的程序适用于气流 1.10。*并且将允许您从气流 2 中带任何操作员。*适应您现有的气流。[气流文件](https://airflow.readthedocs.io/en/latest/backport-providers.html)中给出了这些背端口组件的全部细节。

假设您的 Composer 实例名为`<COMPOSER_INSTANCE>`，位于区域`<COMPOSER_REGION>`，并且您拥有更新该实例的权限，要在 Cloud Composer 中安装一个 backport 包，只需运行:

```
gcloud composer environments update --location=<COMPOSER_REGION>       --update-pypi-package=apache-airflow-backport-providers-google <COMPOSER_INSTANCE>
```

更新将需要几分钟时间。此后，任何新的 DAG 运行都将有新的操作符可用。

**如何检查更新是否成功？**

更新之后，您可能会想，更新真的成功了吗？您需要[连接到 worker 并检查已安装的 Python 依赖项](https://cloud.google.com/composer/docs/how-to/using/installing-python-dependencies#viewing_installed_python_packages)。一旦你创建了必要的`kubectl` 配置，找出一个气流工人舱的名字。首先，检查气流工作者运行的名称空间，用

```
kubectl get namespaces
```

您将得到与此类似的输出(在**粗体**中，Airflow worker pods 运行的名称空间):

```
NAME                                      STATUS AGE
**composer-1–10–4-airflow-1–10–6-f15cbe22**   Active 160d
default                                   Active 160d
kube-node-lease                           Active 160d
kube-public                               Active 160d
kube-system                               Active 160d
```

现在让我们获取该名称空间中的 pod 名称(根据您的名称空间更改*composer-1–10–4-air flow-1–10–6-f15 CBE 22*)。

```
kubectl -n composer-1–10–4-airflow-1–10–6-f15cbe22 get pods
```

您将得到如下列表。我们需要记下其中一个 worker pod 名称:

```
 NAME                                 READY STATUS  RESTARTS AGE
airflow-scheduler-6b5cff877f-j9l4t   2/2   Running 0        27m 
airflow-worker-d977fffc4–6l9vb       2/2   Running 0        25s
airflow-worker-d977fffc4-kktz2       2/2   Running 0        27s
```

现在我们可以连接到一个工人:

```
kubectl exec -itn <COMPOSER_NAMESPACE> <WORKER_NAME> --/bin/bash
```

其中`<WORKER_NAME>`可以是上面列表中的*air flow-worker-d 977 fffc 4–6l9vb*或*air flow-worker-d 977 fffc 4-KKT z2*，在本例中`<COMPOSER_NAMESPACE>`是*composer-1–10–4-air flow-1–10–6-f15 CBE 22*。

然后，您可以检查我们刚刚安装的新依赖项:

```
pip freeze | grep airflow-backport
```

您应该得到与下面类似的输出(可能是更新的版本，这取决于您何时运行该命令):

*Apache-air flow-backport-providers-Google = = 2020 . 11 . 13*

Cloud Composer 为 Apache 气流提供托管服务。当您创建 Composer 实例时，您将被固定到特定的 Airflow 版本，除非您对整个集群进行升级。但是，为了享受最新版本的 Google Cloud 操作符，您不需要更新 Airflow 版本:您可以将最新的操作符从 Airflow 2.x 反向移植到任何使用 Airflow 1.10.x 的 Composer 实例。