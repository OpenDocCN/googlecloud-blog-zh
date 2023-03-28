# 使用谷歌云功能进行图像处理

> 原文：<https://medium.com/google-cloud/image-processing-using-google-cloud-functions-26bffc290c9e?source=collection_archive---------0----------------------->

在这篇博客中，我将分享如何使用谷歌云功能创建一个微服务。目标是当图像上传到谷歌云存储桶时重新格式化图像。

# 设置

我使用谷歌云壳运行所有的命令。关于谷歌云壳的详细信息，请关注[官方文档](https://cloud.google.com/shell/docs/)。

# 解决方案:

据[谷歌](https://cloud.google.com/storage)，

> 云存储提供全球范围内高度耐用的对象存储，可扩展至数十亿字节的数据。您可以从任何存储类别中即时访问数据，通过一个统一的 API 将存储集成到您的应用程序中，并轻松优化价格和性能。

**步骤 1:** 使用 gcloud 命令创建两个 GCS buckets。一个用于源图像序列，另一个用于处理后的图像序列。

```
export SOURCE_BUCKET_NAME=$DEVSHELL_PROJECT_ID-source
export PROCESSED_BUCKET_NAME=$DEVSHELL_PROJECT_ID-processedecho "Creating source bucket: gs://$SOURCE_BUCKET_NAME"
gsutil mb gs://$SOURCE_BUCKET_NAMEecho "Creating processed bucket: gs://$PROCESSED_BUCKET_NAME"
gsutil mb gs://$PROCESSED_BUCKET_NAME
```

**第二步:**使用 gcloud 创建一个谷歌云功能。

> Cloud Functions 是 Google Cloud 的事件驱动的无服务器计算平台。在本地或云中运行您的代码，而无需供应服务器。借助持续交付和监控工具，从编码到部署。云功能可伸缩，因此您只需为使用的计算资源付费。通过连接现有的 Google Cloud 或第三方服务，轻松创建端到端的复杂开发场景。

现在，我们将创建一个 [google cloud 函数](https://cloud.google.com/functions/docs/)，当图像上传到存储器时，它将触发。

云函数的入口点必须在名为`main.py`的 Python 源文件中定义。有关 Python 运行时的详细信息，请点击查看[。给`main.py`增加以下功能。](https://cloud.google.com/functions/docs/concepts/python-runtime#source_code_structure)

```
import os
import cv2
import numpy as np
from google.cloud import storage
from tempfile import NamedTemporaryFile def reformat_image(event, context):
    """Triggered by a change to a Cloud Storage bucket.
    Args:
         event (dict): Event payload.
         context (google.cloud.functions.Context): Metadata for the event.
    """
    file = event client = storage.Client()
    source_bucket = client.get_bucket(file['bucket'])
    source_blob = source_bucket.get_blob(file['name']) image = np.asarray(bytearray(source_blob.download_as_string()), dtype="uint8")
    image = cv2.imdecode(image, cv2.IMREAD_UNCHANGED) scale_percent = 50  # percent of original size
    width = int(image.shape[1] * scale_percent / 100)
    height = int(image.shape[0] * scale_percent / 100)
    dim = (width, height) # resize image
    resized = cv2.resize(image, dim, interpolation=cv2.INTER_AREA) with NamedTemporaryFile() as temp:
        # Extract name to the temp file
        temp_file = "".join([str(temp.name), file['name']]) # Save image to temp file
        cv2.imwrite(temp_file, resized) # Uploading the temp image file to the bucket
        dest_filename = file['name']
        dest_bucket_name = os.environ.get('PROCESSED_BUCKET_NAME', 'Specified environment variable is not set.')
        dest_bucket = client.get_bucket(dest_bucket_name)
        dest_blob = dest_bucket.blob(dest_filename)
        dest_blob.upload_from_filename(temp_file)
```

增加`requirements.txt`，内容如下:

```
opencv-python==4.2.0.34
numpy==1.18.2
google-cloud-storage==1.27.0
```

**第三步:**部署功能

```
gcloud functions deploy reformat_image --set-env-vars PROCESSED_BUCKET_NAME=$PROCESSED_BUCKET_NAME --runtime python37 --trigger-resource $SOURCE_BUCKET_NAME --trigger-event google.storage.object.finalize
```

上述命令将部署该功能。该函数将在创建存储对象时触发。有关云函数触发器的更多详情，请参见[官方文档](https://cloud.google.com/functions/docs/calling/storage)。

现在，我们将复制一个映像到 bucket 来测试我们最近部署的函数。要复制映像，请运行下面提到的命令:

```
gsutil cp /path/to/images gs://$SOURCE_BUCKET_NAME
```

上面提到的命令应该在存储桶中创建一个新对象。这个事件应该会触发云功能。为了验证这一点，我们将运行以下命令并查看日志:

```
gcloud functions logs read reformat_image --limit 50
```

现在，让我们列出来自目的地桶的输出图像

```
gsutil ls gs://$PROCESSED_BUCKET_NAME
```

# 摘要

在这篇博客中，我使用谷歌云平台托管服务来调整图像大小，而没有部署任何服务器。为了避免 GCP 的任何费用，请确保删除 GCS 桶和云功能。

希望这个博客能帮助你理解云的功能和它的用处。

请在 [LinkedIn](https://uk.linkedin.com/in/dheerajbhadani) 或 [Twitter](https://twitter.com/dheerajbhadani) 上分享您的反馈。