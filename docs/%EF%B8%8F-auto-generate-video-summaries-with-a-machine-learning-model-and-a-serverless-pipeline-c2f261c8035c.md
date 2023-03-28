# 用 300 行代码总结视频

> 原文：<https://medium.com/google-cloud/%EF%B8%8F-auto-generate-video-summaries-with-a-machine-learning-model-and-a-serverless-pipeline-c2f261c8035c?source=collection_archive---------0----------------------->

# ⏳2021 年 10 月 8 日更新

*   更新 [**GitHub 版本**](https://github.com/PicardParis/cherry-on-py/tree/main/gcf_video_summary) 最新库版本+ Python 3.7 → 3.9

# 👋你好！

亲爱的开发者们:

你喜欢谚语“一图胜千言”吗？我愿意。我们来看看对*“一图抵千帧”*是否也适用。

在本教程中，您将看到以下内容:

*   如何在一瞬间理解一个视频的内容，
*   在不到 300 行的 Python (3.7)代码中。

![](img/e17571674b294b925c87d0790c527c29.png)

由 35 个序列(镜头)组成的 2'42 "视频生成的视觉摘要。摘要是一个网格，其中每个单元格是一帧，代表一个视频镜头。

# 🔭目标

本教程有两个目标，一个是实践目标，一个是技术目标:

1.  自动生成视频的可视摘要
2.  使用以下属性构建处理管道:

*   托管(随时准备就绪，易于设置)
*   可扩展(能够并行摄取多个视频)
*   不使用时不花费任何东西

# 🛠️工具

一些工具就足够了:

*   视频和结果的存储空间
*   运行代码的无服务器解决方案
*   用于分析视频的机器学习模型
*   从视频中提取帧的库
*   生成可视摘要的库

# 🧱建筑

下面是使用 3 个谷歌云服务([云存储](https://cloud.google.com/storage/docs)、[云功能](https://cloud.google.com/functions/docs)和[视频智能 API](https://cloud.google.com/video-intelligence/docs) )的可能架构:

![](img/75b7a4f08bd6c4006acc7f56e9abf21d.png)

处理管道遵循以下步骤:

1.  您将视频上传到第一个存储桶(存储桶是云中的存储空间)
2.  上传事件自动触发第一个功能
3.  该函数向视频智能 API 发送请求以检测镜头
4.  视频智能 API 分析视频并将结果(注释)上传到第二个存储桶
5.  上传事件触发第二个功能
6.  该功能下载注释和视频文件
7.  该函数将摘要呈现并上传到第三个存储桶
8.  视频摘要准备好了！

# 🐍Python 库

开源客户端库让您可以用惯用的 Python 与 Google 云服务进行交互。您将使用以下内容:

`Cloud Storage`

*   管理下载和上传
*   [https://pypi.org/project/google-cloud-storage](https://pypi.org/project/google-cloud-storage)

`Video Intelligence API`

*   来分析视频
*   [https://pypi.org/project/google-cloud-videointelligence](https://pypi.org/project/google-cloud-videointelligence)

这里有两个额外的 Python 库可供选择，以满足图形需求:

`OpenCV`

*   提取视频帧
*   甚至还有一个无头版本(没有 GUI 特性)，这是服务的理想选择
*   [https://pypi.org/project/opencv-python-headless](https://pypi.org/project/opencv-python-headless)

`Pillow`

*   为了生成视觉摘要
*   `Pillow`是一个非常流行的图像库，既广泛又易于使用
*   [https://pypi.org/project/Pillow](https://pypi.org/project/Pillow)

# ⚙️项目设置

假设你有一个 Google Cloud 账户，你可以用`gcloud`和`gsutil`命令从 Cloud Shell 中设置架构。这让您可以以可重复的方式从头开始编写所有内容。

## 环境变量

```
# Project
**PROJECT_NAME="Visual Summary"
PROJECT_ID="visual-summary-REPLACE_WITH_UNIQUE_SUFFIX"** # Cloud Storage region (https://cloud.google.com/storage/docs/locations)
**GCS_REGION="europe-west1"** # Cloud Functions region (https://cloud.google.com/functions/docs/locations)
**GCF_REGION="europe-west1"** # Source
**GIT_REPO="cherry-on-py"
PROJECT_SRC=~/$PROJECT_ID/$GIT_REPO/gcf_video_summary** # Cloud Storage buckets (environment variables)
**export VIDEO_BUCKET="b1-videos_${PROJECT_ID}"
export ANNOTATION_BUCKET="b2-annotations_${PROJECT_ID}"
export SUMMARY_BUCKET="b3-summaries_${PROJECT_ID}"**
```

> 注意:您可以使用您的 GitHub 用户名作为唯一的后缀。

## 新项目

```
**gcloud projects create $PROJECT_ID \
  --name="$PROJECT_NAME" \
  --set-as-default**Create in progress for [https://cloudresourcemanager.googleapis.com/v1/projects/PROJECT_ID].
Waiting for [operations/cp...] to finish...done.
Enabling service [cloudapis.googleapis.com] on project [PROJECT_ID]...
Operation "operations/acf..." finished successfully.
Updated property [core/project] to [PROJECT_ID].
```

## 账单账户

```
# Link project with billing account (single account)
**BILLING_ACCOUNT=$(gcloud beta billing accounts list \
    --format 'value(name)')**
# Link project with billing account (specific one among multiple accounts)
BILLING_ACCOUNT=$(gcloud beta billing accounts list  \
    --format 'value(name)' \
    --filter "displayName='My Billing Account'")

**gcloud beta billing projects link $PROJECT_ID --billing-account $BILLING_ACCOUNT**billingAccountName: billingAccounts/XXXXXX-YYYYYY-ZZZZZZ
billingEnabled: true
name: projects/PROJECT_ID/billingInfo
projectId: PROJECT_ID
```

## 大量

```
# Create buckets with uniform bucket-level access
**gsutil mb -b on -c regional -l $GCS_REGION gs://$VIDEO_BUCKET
gsutil mb -b on -c regional -l $GCS_REGION gs://$ANNOTATION_BUCKET
gsutil mb -b on -c regional -l $GCS_REGION gs://$SUMMARY_BUCKET**Creating gs://VIDEO_BUCKET/...
Creating gs://ANNOTATION_BUCKET/...
Creating gs://SUMMARY_BUCKET/...
```

你可以在[云控制台](https://console.cloud.google.com/storage/browser)中查看它的样子:

![](img/4d2428a17571f55b249c7e29bad09308.png)

## 服务帐户

创建服务帐户。这仅用于开发目的(生产不需要)。这为您提供了在本地运行代码的凭据。

```
**mkdir ~/$PROJECT_ID
cd ~/$PROJECT_ID

SERVICE_ACCOUNT_NAME="dev-service-account"
SERVICE_ACCOUNT="${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME
gcloud iam service-accounts keys create ~/$PROJECT_ID/key.json --iam-account $SERVICE_ACCOUNT**Created service account [SERVICE_ACCOUNT_NAME].
created key [...] of type [json] as [~/PROJECT_ID/key.json] for [SERVICE_ACCOUNT]
```

设置`GOOGLE_APPLICATION_CREDENTIALS`环境变量，并检查它是否指向服务帐户密钥。当您在当前 shell 会话中运行应用程序代码时，客户端库将使用这些凭据进行身份验证。如果打开新的 shell 会话，请再次设置该变量。

```
**export GOOGLE_APPLICATION_CREDENTIALS=~/$PROJECT_ID/key.json
cat $GOOGLE_APPLICATION_CREDENTIALS**{
  "type": "service_account",
  "project_id": "PROJECT_ID",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "client_email": "SERVICE_ACCOUNT",
  ...
}
```

授权服务帐户访问存储桶:

```
**IAM_BINDING="serviceAccount:${SERVICE_ACCOUNT}:roles/storage.objectAdmin"
gsutil iam ch $IAM_BINDING gs://$VIDEO_BUCKET
gsutil iam ch $IAM_BINDING gs://$ANNOTATION_BUCKET
gsutil iam ch $IAM_BINDING gs://$SUMMARY_BUCKET**
```

## 蜜蜂

默认情况下，会启用一些 API:

```
**gcloud services list**NAME                              TITLE
bigquery.googleapis.com           BigQuery API
bigquerystorage.googleapis.com    BigQuery Storage API
cloudapis.googleapis.com          Google Cloud APIs
clouddebugger.googleapis.com      Cloud Debugger API
cloudtrace.googleapis.com         Cloud Trace API
datastore.googleapis.com          Cloud Datastore API
logging.googleapis.com            Cloud Logging API
monitoring.googleapis.com         Cloud Monitoring API
servicemanagement.googleapis.com  Service Management API
serviceusage.googleapis.com       Service Usage API
sql-component.googleapis.com      Cloud SQL
storage-api.googleapis.com        Google Cloud Storage JSON API
storage-component.googleapis.com  Cloud Storage
```

启用视频智能和云功能 API:

```
**gcloud services enable \
  videointelligence.googleapis.com \
  cloudfunctions.googleapis.com**Operation "operations/acf..." finished successfully.
```

## 源代码

检索源代码:

```
**cd ~/$PROJECT_ID
git clone** [**https://github.com/PicardParis/$GIT_REPO.git**](https://github.com/PicardParis/$GIT_REPO.git)Cloning into 'GIT_REPO'...
...
```

# 🧠视频分析

## 视频镜头检测

视频智能 API 是一个预先训练好的机器学习模型，可以分析视频。多个功能之一是视频镜头检测。对于第一个云函数，这里有一个调用`annotate_video()`的核心函数，带有`SHOT_CHANGE_DETECTION`特性:

## 本地开发和测试

在部署该功能之前，您需要开发和测试它。创建一个 Python 3 虚拟环境并激活它:

```
**cd ~/$PROJECT_ID
python3 -m venv venv
source venv/bin/activate**
```

安装依赖项:

```
**pip install -r $PROJECT_SRC/gcf1_detect_shots/requirements.txt**
```

检查依赖关系:

```
**pip list**Package                        Version
------------------------------ ----------
...
google-cloud-storage           1.28.1
google-cloud-videointelligence 1.14.0
...
```

您可以使用主作用域在脚本模式下测试函数:

> 注意:您已经在 shell 会话中导出了`ANNOTATION_BUCKET`环境变量；您还将在稍后的部署阶段定义它。这使得代码变得通用，并允许您独立于输出桶重用它。

测试功能:

```
**VIDEO_PATH="****cloud-samples-data/video****/gbikes_dinosaur.mp4"
VIDEO_URI="gs://$VIDEO_PATH"
python $PROJECT_SRC/gcf1_detect_shots/main.py $VIDEO_URI**Launching shot detection for <gs://cloud-samples-data/video/gbikes_dinosaur.mp4>...
```

> 注意:测试视频`<gbikes_dinosaur.mp4>`位于外部桶中。这是可行的，因为视频是公开的。

稍等片刻，检查注释是否已经生成:

```
**gsutil ls -r gs://$ANNOTATION_BUCKET**964  YYYY-MM-DDThh:mm:ssZ  gs://ANNOTATION_BUCKET/VIDEO_PATH.json
TOTAL: 1 objects, 964 bytes (964 B)
```

检查注释文件的最后 200 个字节:

```
**gsutil cat -r -200 gs://$ANNOTATION_BUCKET/$VIDEO_PATH.json**}
    }, {
      "start_time_offset": {
        "seconds": 28,
        "nanos": 166666000
      },
      "end_time_offset": {
        "seconds": 42,
        "nanos": 766666000
      }
    } ]
  } ]
}
```

> 注意:这些是最后一个视频镜头的开始和结束位置。一切似乎都很好。

完成后清理干净:

```
**gsutil rm gs://$ANNOTATION_BUCKET/$VIDEO_PATH.json

deactivate

rm -rf venv**
```

## 函数入口点

> 注意:每当一个视频被上传到定义为触发器的桶时，这个函数将被调用。

## 功能部署

部署第一个功能:

```
**GCF_NAME="gcf1_detect_shots"
GCF_SOURCE="$PROJECT_SRC/gcf1_detect_shots"
GCF_ENTRY_POINT="gcf_detect_shots"
GCF_TRIGGER_BUCKET="$VIDEO_BUCKET"
GCF_ENV_VARS="ANNOTATION_BUCKET=$ANNOTATION_BUCKET"
GCF_MEMORY="128MB"

gcloud functions deploy $GCF_NAME \
  --runtime python37  \
  --source $GCF_SOURCE \
  --entry-point $GCF_ENTRY_POINT \
  --update-env-vars $GCF_ENV_VARS \
  --trigger-bucket $GCF_TRIGGER_BUCKET \
  --region $GCF_REGION \
  --memory $GCF_MEMORY \
  --quiet**
```

> 注意:为云函数分配的默认内存是 256MB(可能的值是 128MB、256 MB、512MB、1024MB 和 2048MB)。由于该函数没有内存或 CPU 需求(它发送一个简单的 API 请求)，最小内存设置就足够了。

```
Deploying function (may take a while - up to 2 minutes)...done.
availableMemoryMb: 128
entryPoint: gcf_detect_shots
environmentVariables:
  ANNOTATION_BUCKET: b2-annotations...
eventTrigger:
  eventType: google.storage.object.finalize
...
status: ACTIVE
timeout: 60s
updateTime: 'YYYY-MM-DDThh:mm:ss.mmmZ'
versionId: '1'
```

> 注意:`ANNOTATION_BUCKET`环境变量是用`--update-env-vars`标志定义的。使用环境变量可以让您用不同的触发器和输出桶部署完全相同的代码。

下面是它在[云控制台](https://console.cloud.google.com/functions/list)中的样子:

![](img/99791706a19abca834dead8b31894010.png)

## 生产测试

确保在生产中测试该功能。将视频拷贝到视频桶中:

```
**VIDEO_NAME="gbikes_dinosaur.mp4"
SRC_URI="gs://****cloud-samples-data/video****/$VIDEO_NAME"
DST_URI="gs://$VIDEO_BUCKET/$VIDEO_NAME"

gsutil cp $SRC_URI $DST_URI**Copying gs://cloud-samples-data/video/gbikes_dinosaur.mp4 [Content-Type=video/mp4]...
- [1 files][ 62.0 MiB/ 62.0 MiB]
Operation completed over 1 objects/62.0 MiB.
```

查询日志以检查该功能是否已被触发:

```
**gcloud functions logs read --region $GCF_REGION**LEVEL  NAME               EXECUTION_ID  TIME_UTC  LOG
D      gcf1_detect_shots  ...           ...       Function execution started
I      gcf1_detect_shots  ...           ...       Launching shot detection for <gs://VIDEO_BUCKET/VIDEO_NAME>...
D      gcf1_detect_shots  ...           ...       Function execution took 874 ms, finished with status: 'ok'
```

稍等片刻，检查注释存储桶:

```
**gsutil ls -r gs://$ANNOTATION_BUCKET**
```

您应该会看到注释文件:

```
gs://ANNOTATION_BUCKET/VIDEO_BUCKET/:
gs://ANNOTATION_BUCKET/VIDEO_BUCKET/VIDEO_NAME.json
```

第一个功能是可操作的！

# 🎞️视觉摘要

## 代码结构

有趣的是将代码分成两个主要的类:

*   `StorageHelper`用于本地文件和云存储对象管理
*   `VideoProcessor`用于图形处理

下面是一个可能的核心函数:

> 注意:如果出现异常，用`logging.exception()`记录它们以获得生产日志中的堆栈跟踪是很方便的。

## 类别`StorageHelper`

该类管理以下内容:

*   视频镜头注释的检索和解析
*   源视频的下载
*   上传生成的可视摘要
*   文件名

源视频在`with`语句上下文管理器中处理:

> 注:下载后，视频将使用`/tmp` RAM 磁盘中的存储空间(无服务器功能的唯一可写空间)。最好在不再需要临时文件时将其删除，以避免将来调用该函数时出现潜在的内存不足错误。

使用方法`storage.Blob.download_as_string()`和`json.loads()`检索注释:

解析由这个`VideoShot`助手类处理:

视频拍摄信息可以通过 getter 和 generator 公开:

选择命名约定是为了在不同的存储桶之间保持一致的对象路径。这也允许您从注释 URI 中推断出视频路径:

视频直接用`storage.Blob.download_to_filename()`下载:

相反，可以使用`storage.Blob.upload_from_string()`上传结果:

> 注意:`from_string`在这里表示`from_bytes`(Python 2 遗留)。`Pillow`支持使用内存镜像，避免了管理本地文件。

最后，这里是摘要文件的一个可能的命名约定:

## 类别`VideoProcessor`

该类管理以下内容:

*   视频帧提取
*   可视化摘要生成

打开和关闭视频在`with`语句上下文管理器中处理:

视频摘要是一个单元格网格，可以用两个发生器在单个循环中渲染:

> 注意:`shot_ratio`默认设置为`0.5`，提取视频拍摄的中间帧。

第一个生成器生成细胞图像:

第二个生成器产生单元格位置:

`OpenCV`轻松提取给定位置的视频帧:

选择摘要网格构成是任意的。下面是一个示例，用于编写保留视频比例的摘要:

最后，`Pillow`提供了对图像序列化的完全控制:

> 注意:使用内存中的映像可以避免管理本地文件，并使用较少的内存。

## 本地开发和测试

您可以使用主作用域在脚本模式下测试函数:

测试功能:

```
**cd ~/$PROJECT_ID
python3 -m venv venv
source venv/bin/activate

pip install -r $PROJECT_SRC/gcf2_generate_summary/requirements.txt

VIDEO_NAME="gbikes_dinosaur.mp4"
ANNOTATION_URI="gs://$ANNOTATION_BUCKET/$VIDEO_BUCKET/$VIDEO_NAME.json"

python $PROJECT_SRC/gcf2_generate_summary/main.py $ANNOTATION_URI**Downloading -> /tmp/SUMMARY_BUCKET/VIDEO_BUCKET/VIDEO_NAME
Generating summary...
Uploading -> VIDEO_BUCKET/VIDEO_NAME.summary004.jpeg
```

> 注:上传的视频摘要显示 4 个镜头。

清理:

```
**deactivate
rm -rf venv**
```

## 函数入口点

> 注意:每当一个注释文件被上传到定义为触发器的桶时，这个函数就会被调用。

## 功能部署

```
**GCF_NAME="gcf2_generate_summary"
GCF_SOURCE="$PROJECT_SRC/gcf2_generate_summary"
GCF_ENTRY_POINT="gcf_generate_summary"
GCF_TRIGGER_BUCKET="$ANNOTATION_BUCKET"
GCF_ENV_VARS="SUMMARY_BUCKET=$SUMMARY_BUCKET"
GCF_TIMEOUT="540s"
GCF_MEMORY="512MB"

gcloud functions deploy $GCF_NAME \
  --runtime python37  \
  --source $GCF_SOURCE \
  --entry-point $GCF_ENTRY_POINT \
  --update-env-vars $GCF_ENV_VARS \
  --trigger-bucket $GCF_TRIGGER_BUCKET \
  --region $GCF_REGION \
  --timeout $GCF_TIMEOUT \
  --memory $GCF_MEMORY \
  --quiet**
```

注意事项:

*   云函数的默认超时是 60 秒。当您部署一个可能需要长时间处理的后台函数时，将其设置为最大值(540 秒= 9 分钟)。
*   你还需要为视频和图像处理增加一点内存。根据视频的大小和输出摘要的最大分辨率，或者如果您需要更快地生成摘要(内存大小和 vCPU 速度相关)，您可以使用更高的值(1024MB 或 2048MB)。

```
Deploying function (may take a while - up to 2 minutes)...done.
availableMemoryMb: 512
entryPoint: gcf_generate_summary
environmentVariables:
  SUMMARY_BUCKET: b3-summaries...
...
status: ACTIVE
timeout: 540s
updateTime: 'YYYY-MM-DDThh:mm:ss.mmmZ'
versionId: '1'
```

下面是它在[云控制台](https://console.cloud.google.com/functions/list)中的样子:

![](img/09759d1e834a26df4ec881a97442992e.png)

## 生产测试

确保在生产中测试该功能。您可以在第二个存储桶中上传注释文件:

```
**VIDEO_NAME="gbikes_dinosaur.mp4"
ANNOTATION_FILE="$VIDEO_NAME.json"
ANNOTATION_URI="gs://$ANNOTATION_BUCKET/$VIDEO_BUCKET/$ANNOTATION_FILE"
gsutil cp $ANNOTATION_URI .
gsutil cp $ANNOTATION_FILE $ANNOTATION_URI
rm $ANNOTATION_FILE**
```

> 注意:这将重用先前的本地测试注释文件并覆盖它。覆盖桶中的文件也会触发附加的函数。

等待几秒钟，并查询日志以检查该功能是否已被触发:

```
**gcloud functions logs read --region $GCF_REGION**LEVEL  NAME                   EXECUTION_ID  TIME_UTC  LOG
...
D      gcf2_generate_summary  ...           ...       Function execution started
I      gcf2_generate_summary  ...           ...       Downloading -> /tmp/SUMMARY_BUCKET/VIDEO_BUCKET/VIDEO_NAME
I      gcf2_generate_summary  ...           ...       Generating summary...
I      gcf2_generate_summary  ...           ...       Uploading -> VIDEO_BUCKET/VIDEO_NAME.summary004.jpeg
D      gcf2_generate_summary  ...           ...       Function execution took 11591 ms, finished with status: 'ok'
```

第二个功能正在运行，管道已经到位！现在，您可以通过在第一个桶中复制新视频来进行端到端测试。

## 结果

将生成的摘要下载到您的计算机上:

```
**cd ~/$PROJECT_ID
gsutil cp -r gs://$SUMMARY_BUCKET/**.jpeg .
cloudshell download *.jpeg**
```

以下是`gbikes_dinosaur.mp4` (4 个检测到的镜头)的视觉摘要:

![](img/a4e29826d6cbe5cf2e4521c8049142e1.png)

您也可以从[云控制台](https://console.cloud.google.com/storage/browser/)直接预览文件:

![](img/1eeb933ad7ad66a345d3c01646a1b1d4.png)

# 🍒Py 上的樱桃🐍

现在，蛋糕上的糖衣(或者我们用法语说的“馅饼上的樱桃”)…

基于相同的架构和代码，您可以添加一些功能:

*   触发其他桶中视频的处理
*   生成多种格式的摘要(如 JPEG、PNG、WEBP)
*   生成动画摘要(也有多种格式，如 GIF、PNG、WEBP)

丰富架构以复制 2 个项目:

*   视频镜头检测功能，使其作为 HTTP 端点运行
*   处理动画图像的摘要生成功能

调整代码以支持新功能:

*   用于生成静态或动态摘要的`animated`参数
*   以多种格式保存和上传结果

## 架构(第 2 版)

![](img/8840f846967d8cbe723fd3fa5487cb30.png)

*   A.视频镜头检测也可以通过 HTTP GET 请求手动触发
*   B.静态和动态摘要在两个函数中并行生成
*   C.摘要以多种图像格式上传

## HTTP 入口点

> 注意:这是与`gcf_detect_shots`相同的代码，带有从 GET 请求中提供的视频 URI 参数。

## 功能部署

```
GCF_NAME="gcf1_detect_shots_http"
GCF_SOURCE="$PROJECT_SRC/gcf1_detect_shots"
GCF_ENTRY_POINT="gcf_detect_shots_http"
GCF_TRIGGER_BUCKET="$VIDEO_BUCKET"
GCF_ENV_VARS="ANNOTATION_BUCKET=$ANNOTATION_BUCKET"
GCF_MEMORY="128MB"

gcloud functions deploy $GCF_NAME \
  --runtime python37  \
  --source $GCF_SOURCE \
  --entry-point $GCF_ENTRY_POINT \
  --update-env-vars $GCF_ENV_VARS \
  --trigger-http \
  --region $GCF_REGION \
  --memory $GCF_MEMORY \
  --quiet
```

下面是它在[云控制台](https://console.cloud.google.com/functions/list)中的样子:

![](img/06d4abe164a2b1a15a1ae305a85e40cf.png)

## 动画支持

在核心功能中增加一个`animated`选项:

定义您有兴趣生成的格式:

添加支持以不同格式生成静态和动态摘要:

序列化仍然可以在单个函数中进行:

> 注意:`Pillow`是通用和一致的，允许重要和干净的代码分解。

向`StorageHelper`类添加一个`animated`可选参数:

最后，在入口点添加一个`ANIMATED`可选环境变量:

## 功能部署

使用附加的`ANIMATED`环境变量复制第二个函数:

```
**GCF_NAME="gcf2_generate_summary_animated"
GCF_SOURCE="$PROJECT_SRC/gcf2_generate_summary"
GCF_ENTRY_POINT="gcf_generate_summary"
GCF_TRIGGER_BUCKET="$ANNOTATION_BUCKET"
GCF_ENV_VARS1="SUMMARY_BUCKET=$SUMMARY_BUCKET"
GCF_ENV_VARS2="ANIMATED=1"
GCF_TIMEOUT="540s"
GCF_MEMORY="2048MB"

gcloud functions deploy $GCF_NAME \
  --runtime python37  \
  --source $GCF_SOURCE \
  --entry-point $GCF_ENTRY_POINT \
  --update-env-vars $GCF_ENV_VARS1 \
  --update-env-vars $GCF_ENV_VARS2 \
  --trigger-bucket $GCF_TRIGGER_BUCKET \
  --region $GCF_REGION \
  --timeout $GCF_TIMEOUT \
  --memory $GCF_MEMORY \
  --quiet**
```

下面是它在[云控制台](https://console.cloud.google.com/functions/list)中的样子:

![](img/aff146adcee461ba070dd13060c6af21.png)

# 🎉最终测试

HTTP 端点允许您用 GET 请求触发管道:

```
**GCF_NAME="gcf1_detect_shots_http"
VIDEO_URI="gs://****cloud-samples-data/video****/visionapi.mp4"
GCF_URL="https://$GCF_REGION-$PROJECT_ID.cloudfunctions.net/$GCF_NAME?video_uri=$VIDEO_URI"

curl $GCF_URL -H "Authorization: bearer $(gcloud auth print-identity-token)"**Launched shot detection for video_uri <VIDEO_URI>
```

> 注意:测试视频`<visionapi.mp4>`位于一个外部存储桶中，但可以公开访问。

此外，将一个或多个视频复制到视频桶中。您可以拖放视频:

![](img/f8f936189d1adc5499dc691f55b5f06f.png)

然后视频被并行处理。以下是一些日志:

```
LEVEL NAME                           EXECUTION_ID ... LOG
...
D     gcf2_generate_summary_animated f6n6tslsfwdu ... Function execution took 49293 ms, finished with status: 'ok'
I     gcf2_generate_summary          yd1vqabafn17 ... Uploading -> b1-videos.../JaneGoodall.mp4.summary035_still.png
I     gcf2_generate_summary_animated qv9b03814jjk ... shot_ratio: 43%
I     gcf2_generate_summary          yd1vqabafn17 ... Uploading -> b1-videos.../JaneGoodall.mp4.summary035_still.webp
D     gcf2_generate_summary          yd1vqabafn17 ... Function execution took 54616 ms, finished with status: 'ok'
I     gcf2_generate_summary_animated g4d2wrzxz2st ... shot_ratio: 71%
...
D     gcf2_generate_summary          amwmov1wk0gn ... Function execution took 65256 ms, finished with status: 'ok'
I     gcf2_generate_summary_animated 7pp882fz0x84 ... shot_ratio: 57%
I     gcf2_generate_summary_animated i3u830hsjz4r ... Uploading -> b1-videos.../JaneGoodall.mp4.summary035_anim.png
I     gcf2_generate_summary_animated i3u830hsjz4r ... Uploading -> b1-videos.../JaneGoodall.mp4.summary035_anim.webp
D     gcf2_generate_summary_animated i3u830hsjz4r ... Function execution took 70862 ms, finished with status: 'ok'
...
```

在第三个桶中，您将找到所有静态和动态摘要:

![](img/ae909eabb7d1e5a94bf452fb10152e80.png)

你已经看到了作为本教程介绍的`<JaneGoodall.mp4>`的静态摘要。在动画版本中，只有 6 帧，你会对[的整个视频](https://storage.googleapis.com/cloud-samples-data/video/JaneGoodall.mp4)有更好的了解:

![](img/8fa83f631fd18de398472e3eb4621037.png)

如果您不想保留您的项目，您可以删除它:

```
**gcloud projects delete $PROJECT_ID**
```

# ➕还有一件事

```
**first_line_after_licence=16
find $PROJECT_SRC -name '*.py' -exec tail -n +$first_line_after_licence {} \; | grep -v "^$" | wc -l**289
```

你用不到 300 行 Python 代码就完成了所有工作。少台词，少 bug！

**🔥🐍任务完成！🐍🔥**

# 🖖再见

我希望你能欣赏这篇教程，并乐意阅读你的反馈。也可以[在 Twitter 上关注我](https://twitter.com/PicardParis)。

# ⏳更新

*   **2021–10–08**:更新 [GitHub 版本](https://github.com/PicardParis/cherry-on-py/tree/main/gcf_video_summary)最新库版本+ Python 3.7 → 3.9

# 📜同样在这个系列中

1.  总结视频
2.  [跟踪视频对象](/google-cloud/video-object-tracking-as-a-service-18eb4227df34?source=friends_link&sk=c9602c33c77aa950a59282b6de5c0c57)
3.  [人脸检测和处理](/google-cloud/face-detection-and-processing-in-300-lines-of-code-38dc51a115d4?source=friends_link&sk=cc252ab86eab9ed2e8583963d0598661)
4.  [处理图像](/google-cloud/deploy-a-coloring-page-generator-in-minutes-with-cloud-run-bff59e59d890?source=friends_link&sk=a3d6e22e7e77828e411592f46025531e)