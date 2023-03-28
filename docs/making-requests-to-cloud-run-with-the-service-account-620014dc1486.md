# 使用服务帐户向云运行发出请求

> 原文：<https://medium.com/google-cloud/making-requests-to-cloud-run-with-the-service-account-620014dc1486?source=collection_archive---------1----------------------->

Cloud Run 是 Google 云平台上新的计算无服务器解决方案。它可以运行任何部署为 Docker image 的 web 应用程序。它的一个很好的功能是内置的自动认证，即你可以隐藏公共互联网的服务，并通过 IAM 控制访问。这样，您可以向具体的用户或组授予访问权限。由于云运行的主要用途之一是微服务，并且通过访问控制功能，可以方便地将其用于内部微服务(您希望它是私有的)，其中一种方法是使用服务帐户。

在[官方文档](https://cloud.google.com/run/docs/securing/authenticating#service-to-service)中，描述了如何使用服务对服务的身份验证，并提供了从 Google Cloud 发出请求的代码示例，其中身份验证凭证是从元数据服务器获得的，因此不需要服务帐户。如果您想要在 GCP 以外的地方请求云运行服务，则需要服务帐户。现在在文档中，有描述如何做的步骤，但是没有代码示例。

所以在这篇文章中，我想描述如何设置私有的云运行服务，以及如何使用服务帐户进行请求。作为一个应用程序，我创建了 Docx 到 PDF 的转换器，类似于它在 Cloud Next '19 keynote 中的演示。服务也可以用作 Pub Sub HTTP 目标，并用于异步处理，我将在下一篇文章中对此进行描述。

这个例子的完整代码在 Github 资源库[https://Github . com/zde nulo/GCP-docx 2 pdf/tree/master/cloud _ run _ pubsub](https://github.com/zdenulo/gcp-docx2pdf/tree/master/cloud_run_pubsub)中。

关于 web 服务，没有什么特别的，Libreoffice 可以通过 Docker 安装和使用，这很酷。Web 服务被定制为接受来自发布订阅的 json 消息，最小 POST 请求需要采用以下格式:

```
{
   "message":{
      "attributes":{
         "bucketId":"BUCKET-ID",
         "objectId":"FILE-PATH"
      }
   }
}
```

服务需要一个需要转换的 Docx 文件存储在云存储中，因此 bucket 和 filename(路径)是必需的输入。

为了构建和部署服务，Cloud Build 与配置文件 cloudbuild.yaml 一起使用

```
steps:
  # build the container image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${TAG_NAME}', '.']
  # push the container image to Container Registry
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}']
  # Deploy container image to Cloud Run
- name: 'gcr.io/cloud-builders/gcloud'
  args: ['beta', 'run', 'deploy', '${_SERVICE_NAME}', '--image', 'gcr.io/$PROJECT_ID/${_SERVICE_NAME}:${TAG_NAME}', '--region', 'us-central1', '--memory', '1Gi', '--update-env-vars', '${_ENV_VARIABLES}']
images:
- gcr.io/$PROJECT_ID/${_SERVICE_NAME}
```

服务期望设置环境变量 OUTPUT_BUCKET(这是保存 PDF 的存储桶的名称),这是在部署期间完成的。此外，还需要定义云运行服务的名称。

使用以下命令启动构建和部署:

```
gcloud builds submit --config=cloudbuild.yaml --substitutions=_SERVICE_NAME="<service name>",TAG_NAME="v0.1",_ENV_VARIABLES="OUTPUT_BUCKET=<name of output bucket>"
```

默认情况下，云运行服务部署为私有。

下一步是创建服务帐户并分配特定的角色。要创建名为 cr-test 的服务帐户，我将执行以下命令:

```
~>gcloud iam service-accounts create cr-test --display-name="Cloud Run Test"
Created service account [cr-test].
```

然后，正如官方文档所说，我将向服务帐户角色添加云运行调用者，这是向云运行服务发出请求所必需的:

```
~> gcloud beta run services add-iam-policy-binding sa-run --member=serviceAccount:cr-test@adventures-on-gcp.iam.gserviceaccount.com --role=roles/run.invoker
Updated IAM policy for service [sa-run].
bindings:
- members:
  - serviceAccount:cr-test@adventures-on-gcp.iam.gserviceaccount.com
  role: roles/run.invoker
etag: BwWHWbhxA2c=
```

另一种方法是向服务帐户添加 IAM 策略绑定。这样，服务帐户将显示在 IAM 部分，如果需要，您可以为其分配多个角色。

```
gcloud projects add-iam-policy-binding <PROJECT-ID> --member=serviceAccount:cr-test@adventures-on-gcp.iam.gserviceaccount.com --role=roles/run.invoker
```

最后一步是创建一个私钥文件(在我的例子中，我将其命名为 cr-test-secret.json ),并将其下载到本地，以便从本地计算机向云运行服务发出请求:

```
gcloud iam service-accounts keys create cr-test-secret.json --iam-account=cr-test@adventures-on-gcp.iam.gserviceaccount.com
```

使用服务帐户凭证在 Python 中发出请求的代码在 api_request.py 文件中，只有几行，BUCKET_NAME 和 API_URL 需要适当设置。

```
from google.oauth2 import service_account
from google.auth.transport.requests import AuthorizedSession

SERVICE_FILENAME = 'cr-test-secret.json'
BUCKET_NAME = ''  # name of the bucket where Docx file is saved
API_URL = ''  # change to url of you Cloud Run service

audience = API_URL
credentials = service_account.IDTokenCredentials.from_service_account_file(SERVICE_FILENAME, target_audience=audience)

session = AuthorizedSession(credentials)

data = {"message": {"attributes": {"bucketId": BUCKET_NAME, "objectId": "demo.docx"}}}

r = session.post(API_URL, json=data)
print(r.text)
```

这里最重要的是要注意使用 service_accounts 模块中的哪个类。我通常使用**credentials . from _ service _ account()**但是在这种情况下，需要使用 **IDTokenCredentials** 类。如文档中所述，两者之间的区别在于“这些凭证在很大程度上类似于凭证类，但是它们不使用 OAuth 2.0 访问令牌作为承载令牌，而是使用 Open ID Connect ID 令牌作为承载令牌。当与需要 ID 令牌但不能接受访问令牌的服务进行通信时，这些凭据非常有用。

AuthorizedSession 基本上是一个“请求”库的包装器，以正确的头发出请求。