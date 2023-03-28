# 如何在 Google Cloud 中启动 Jupyter——Python 方式

> 原文：<https://medium.com/google-cloud/how-to-start-jupyter-in-google-cloud-the-python-way-7a70adb7b4b1?source=collection_archive---------0----------------------->

你知道你可以用 Python SDK 在谷歌云中轻松启动 Jupyter 笔记本吗？还有自动挂载 GCS 桶，加 GPU 还是用自己的容器？

![](img/dc7dfdf4739b4497a696c33b0cc9e9b3.png)

照片由[苏三·李](https://unsplash.com/@blackodc?utm_source=medium&utm_medium=referral)在 [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral) 拍摄

*这是用 Python SDK 操作 GCP 顶点人工智能系列的第一篇文章。订阅获取下一篇博文。*

谷歌于 2021 年 5 月推出了新的机器学习平台 Vertex AI，接替了之前的人工智能平台。他们还发布了多种语言的 SDK。让我们把重点放在 Python 上，因为这将是许多数据科学家的首选。

# 安装 Python SDK

Google 目前为 Vertex AI 服务维护了两个 Python 库:

*   *Google-cloud-notebooks*——用于操作顶点人工智能工作台(又名 GCP 的 Jupyter 笔记本)
*   *谷歌云平台*——用于操控其他一切

所以现在，我们只安装笔记本库:

```
pip install google-cloud-notebooks
```

关于 GCP Python 库的文档有时很难找到，笔记本 SDK 在 https://googleapis.dev/python/notebooks/latest/index.html[有描述。](https://googleapis.dev/python/notebooks/latest/index.html)

# 使用谷歌云授权

我将假设你已经创建了 GCP 项目和顶点人工智能是启用的。如果没有，请遵循[官方文件](https://cloud.google.com/vertex-ai/docs/start/cloud-environment)。

有多种方法可以验证 Google Cloud。出于我们的目的，我们将利用 gcloud CLI 工具。我们将在终端中登录，然后 Python 中的笔记本服务将能够自动获取凭证。

1.  [安装 gcloud CLI 工具](https://cloud.google.com/sdk/docs/install)
2.  [使用您的用户帐户登录 g cloud](https://cloud.google.com/sdk/gcloud/reference/auth/login)

```
gcloud auth login
```

或者，您可以使用服务帐户:

```
gcloud auth activate-service-account --key-file=$KEY_FILE
```

# GCP 权限

为了能够启动和删除笔记本，您需要 **roles/notebooks.runner** 。但是，删除笔记本需要**角色/笔记本.管理**。

# Python 笔记本服务

谷歌提供了两种操作笔记本的方式——异步客户端和同步客户端。选择哪一个完全由你决定。为了简单起见，我将在本教程的剩余部分使用同步客户端。两者都记载了[这里的](https://googleapis.dev/python/notebooks/latest/notebooks_v1/notebook_service.html)。

现在，我们可以启动笔记本电脑服务客户端:

```
from google.cloud import notebooks_v1client = notebooks_v1.NotebookServiceClient()
```

# 定义笔记本

有趣的是。在定义您的笔记本电脑环境和在预算允许的情况下增加尽可能多的计算能力方面，您有许多可能性。

## 选择环境和机器类型

你可以基于谷歌为你提供的虚拟机镜像来创建你的笔记本，也可以使用你的 docker 镜像。我将在接下来的一些文章中讨论为笔记本构建映像，所以让我们首先关注 Google 提供的虚拟机映像。

GCP 有多个深度学习虚拟机映像可供使用，为了方便起见，其中许多还附带了预安装的 Jupyter 和已配置的端口设置。他们有 Tensorflow、Pytorch、R 和其他框架的图片。完整的名单可以在[这里](https://cloud.google.com/vertex-ai/docs/workbench/user-managed/images)找到。在下一篇文章中，我将介绍如何轻松地建立一个 Julia 笔记本。

假设我们要用 Tensorflow 2.8 加 GPU 支持，我会选择 VM 镜像族`tf-ent-2-8-cu113-notebooks`。

Vertex AI Workbench 只是一个运行 Jupyter 的计算引擎实例。您可以在[文档](https://cloud.google.com/compute/docs/machine-types)中找到关于可用计算引擎机器类型的更多信息。出于我们的教程目的，我将选择`n1-standard-8`——具有 8 个 vCPUs 和 32 GB 内存的标准机器类型。然而，你可以随心所欲地疯狂。

## GPU 和磁盘

关于机器类型和位置，GPU 的使用有许多[限制](https://cloud.google.com/compute/docs/gpus)。我会选择英伟达特斯拉 P100。在 GPU 上选择阅读[谷歌页面。也有可能附加更多的 GPU(通常是 1，2，4 或 8)，但我只会用一个。](https://cloud.google.com/compute/docs/gpus)

当你在 GCP 创建一个笔记本时，谷歌会为你创建两个磁盘。

1.  引导磁盘—操作系统/库/初始化脚本所在的位置
2.  数据磁盘—映射到/home/jupyter 文件夹

对于每一种，您可以在标准/平衡/ SSD 持久磁盘之间进行选择。每个大小从 100GB 到 64 TB 不等。为了简单起见，我们可以为这两个选项选择 200GB 的平衡磁盘。

## 创建笔记本请求

根据我们已有的信息，我们可以像这样创建笔记本请求:

```
from google.cloud.notebooks_v1.types import Instance, VmImage

notebook_instance = Instance(
    vm_image=VmImage(
        project="deeplearning-platform-release",
        image_family="tf-ent-2-8-cu113-notebooks",
    ),
    machine_type="n1-standard-8",
    accelerator_config=Instance.AcceleratorConfig(
        type_=Instance.AcceleratorType.NVIDIA_TESLA_P100, core_count=1
    ),
    install_gpu_driver=True,
    boot_disk_type=Instance.DiskType.PD_BALANCED,
    boot_disk_size_gb=200,
    data_disk_type=Instance.DiskType.PD_BALANCED,
    data_disk_size_gb=200,
)
```

这是最基本的东西。您可以添加更多参数，如标签、标记、元数据或实例所有者。进入[此处](https://googleapis.dev/python/notebooks/latest/notebooks_v1/types.html#google.cloud.notebooks_v1.types.Instance)读取所有参数。

## 向 GCP 发送请求

Python SDK 经常使用一种叫做 **parent** 的东西。Parent 只是一个字符串，包含关于您的 GCP 项目和您想要使用的位置的信息。

现在，我们准备将请求发送到 GCP 端点。

```
project_id = "PROJECT_ID" # Put your own project id here
location = "europe-west1-a" # Put your own location here
parent = f"projects/{project_id}/locations/{location}"request = notebooks_v1.CreateInstanceRequest(
        parent=parent,
        instance_id="my-first-notebook",
        instance=instance,
    )
op = client.create_instance(request=request)
op.result()
```

来自`client.create_instance`的结果是 Google 的长期运行操作——[读取文档](https://googleapis.dev/python/google-api-core/latest/operation.html#google.api_core.operation.Operation)。等待操作完成的最简单的方法是方法`op.result()`，它也提供关于创建的笔记本或创建过程中的错误的信息。

以下是完整的代码:

# 技巧 1:使用启动脚本挂载 GCS 存储桶

正如我们之前所说，GCP 上的 Jupyter 只是一个配置的计算引擎(CE)虚拟机。每个 CE 虚拟机都可以有一个[启动脚本](https://cloud.google.com/compute/docs/instances/startup-scripts/linux)——在机器启动过程中执行的 bash 或非 bash 文件。这给了你无限的可能性来提升你的 Jupyter 笔记本电脑。我将只向您展示如何自动安装带有 [gcsfuse](https://github.com/GoogleCloudPlatform/gcsfuse) 的 GCS 铲斗。

请注意，启动脚本是作为`root`用户运行的。但是当你连接到 Jupyter 时，你将成为`jupyter`用户。

gcsfuse 已经预装在谷歌提供的图片中。在其他映像中，您必须自己安装它。

```
#!/bin/bashLOCAL_PATH=/home/jupyter/mounted/gcs
BUCKET_NAME=my-super-bucket # Change this to your bucket
BUCKET_DIR=notebook_data

sudo su -c "mkdir -p $LOCAL_PATH"
sudo su -c "gcsfuse --implicit-dirs --only-dir=$BUCKET_DIR $BUCKET_NAME $LOCAL_PATH"
```

我在这里使用了两个技巧。

*   `—-implicit-dirs`对于目录的隐式存在，这允许你看到桶中的所有对象，但是它有几个缺点，主要是成本更高
*   `—-only-dir`只挂载一个“目录”的桶

该脚本必须保存在 GCS 中或公共可用的 URL 上(如 Github)。我将向您展示如何将一个字符串直接上传到 GCS，并保存为文本，而不保存在本地。

```
pip install google-cloud-storage
```

现在，在发送到 GCP 之前，只需在`notebook_instance`中添加一个参数:

```
notebook_instance.post_startup_script = f"gs://{blob.bucket.name}/{blob.name}"request = notebooks_v1.CreateInstanceRequest(
    parent=parent,
    instance_id="my-first-notebook",
    instance=notebook_instance,
)
op = client.create_instance(request=request)
op.result()
```

# 技巧#2:托管笔记本

谷歌为 Jupyter 笔记本提供了两种解决方案。到目前为止，我们使用的是**用户管理的笔记本**，它允许大量定制。第二个选项是 [**托管笔记本**](https://cloud.google.com/vertex-ai/docs/workbench/notebook-solution#managed) ，在这里你可以先写代码，然后再选择硬件。API 非常相似，你可以在这里找到文档。

# 提示 3:阅读 Lak Lakshmanan 的文章

[如何在 Google Cloud VM 上使用 Jupyter](https://towardsdatascience.com/how-to-use-jupyter-on-a-google-cloud-vm-5ba1b473f4c2)是 Lak Lakshmanan(前 Google)关于在 GCP 上使用 gcloud CLI 工具启动 Jupyter 的一篇优秀文章。它有点旧，但包含一些很棒的技巧。

*请订阅我的频道获取下一篇关于如何使用 Vertex AI Python SDK 的博文。除了展示其他 Vertex AI 服务之外，我将很快发布一个帖子，展示如何通过 Vertex AI Workbench 和云构建轻松启动 Jupyter 和 Julia。*