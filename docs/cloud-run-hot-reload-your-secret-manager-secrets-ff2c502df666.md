# 云运行:热重装您的秘密经理的秘密

> 原文：<https://medium.com/google-cloud/cloud-run-hot-reload-your-secret-manager-secrets-ff2c502df666?source=collection_archive---------1----------------------->

![](img/20ba922f6030dac47c102efb7edb5b08.png)

对于任何类型的服务和环境，安全性都是至关重要的。存在安全地大规模地**托管、管理和提供机密的解决方案。在谷歌云上， [**秘密管理器**](https://cloud.google.com/secret-manager) **是专门负责秘密管理的原生服务**。**

为了简化访问秘密， **Secret Manager 被直接集成**在一些 Google 云服务中，如 Cloud Build、Cloud Functions 和 Cloud Run。

**【了解云跑？查看** [**使用新的专用运行时间检查**](http://bit.ly/CloudRunVerify) **验证云运行服务的可用性。】**

因此，通常的秘密管理需要**改变、轮换和更新秘密**

> 如何在云运行实例中获得最新的秘密版本，没有延迟，不需要重新启动它们？

# 秘密的最佳实践

其中最重要的一条规则是:“**秘密必须保密。为此，秘密管理器服务通过 IAM 服务确保**加密** [静止](https://cloud.google.com/secret-manager/docs/encryption)和[传输](https://cloud.google.com/docs/security/encryption-in-transit)以及**访问许可** [。](https://cloud.google.com/secret-manager/docs/access-control)**

此外，对于长期存在的秘密(如密码或 API 密钥)，建议**定期轮换和更新秘密**(Google Cloud 文档中提到的*关于* [*90 天*](https://cloud.google.com/kms/docs/key-rotation#how_often_to_rotate_keys) *)。并且最好**对应用程序没有任何业务影响**。*

*我的朋友*[*Antoine Castex*](https://medium.com/u/d0e632e7c73a?source=post_page-----ff2c502df666--------------------------------)*[*记录了一个在欧莱雅实现的解决方案*](/google-cloud/the-key-wars-story-a65bb9dabe56) *用于 Google Cloud 上的密钥旋转。**

# *云运行秘密集成*

*Cloud Run 支持与 Secret Manager 服务集成的两种模式。*

*   *将秘密加载到环境变量中*
*   *将秘密载入文件*

## *环境变量求解的局限性*

*Secret Manager 当前最常用的用法是将机密作为环境变量加载。从代码中使用很简单:*

*   *直接访问操作系统环境*
*   *没有要打开、关闭的文件，没有要读取的流*
*   *没有要执行的字节到字符串转换*

*所以，**完美的解决方案开始吧！***

*在云运行部署中，您必须**将一个环境变量名与一个秘密引用**绑定，如下面的示例所示*

```
*gcloud run deploy secret-read \
  --set-secrets=mysecret=medium:latest*
```

*当使用环境变量模式时，**这个秘密在实例启动时被访问。** *这时，执行环境设置了正确的标准和自定义环境变量。**

*因此，**在整个实例生命周期中，这个秘密只被读取一次，不再被读取。
因此，即使秘密发生变化，**当前运行的实例也不会被注意到**并且您不能使用新的秘密。**只有新实例可以**。
*同样，如果你删除了这个秘密，当前的实例并不知道这个删除。****

*您有 **2 种可能的解决方案**来重新加载最新版本的秘密:*

*   *等待旧的实例自动停止。
    云运行不提供**停止正在运行的实例**的能力。在处理完最后一个请求 15 分钟后，实例自动停止[(空闲模式)。如果你遇到持续的交通堵塞，可能需要几个小时，甚至几天！！](https://cloud.google.com/run/docs/container-contract#idle)*
*   ***重新部署**您的云运行服务的新版本。
    除了**部分无效**之外，解决方案**在秘密更新**上自动化起来很复杂。
    当然，新的请求将由新部署的修订版(加载了新的秘密版本)来服务，但是**现有实例继续服务“部署前”流量**(并且可能需要长达 1 小时(最大云运行超时)，特别是当您服务[双向流或 websocket 解决方案](https://cloud.google.com/blog/products/serverless/cloud-run-gets-websockets-http-2-and-grpc-bidirectional-streams)时*

## *文件挂载解决方案*

*将一个**秘密挂载为一个文件**在开始时并不那么自然，即使 Kubernetes 通过 ConfigMap 使这种方式**流行起来。***

*借助云运行，您也可以通过命令行获得这种能力*

```
*gcloud run deploy secret-read \
  --set-secrets=/secret/mysecret=medium:latest*
```

**可以看出，与环境变量模式的区别在于* ***以*** `***/***`开头的名字的前缀*

*这一次，秘密被挂载为一个文件，但是**它不是一个“真实的文件”**。它更像是一个**代理，捕捉秘密访问请求，并动态执行对秘密管理器服务的 API 调用**。*

*因此，每次在代码中读取文件时，都会访问和读取**秘密。
就这样，最新的秘密版本被访问了！***

**只有当您的云运行实例* ***挂载了您的*** `***latest***` ***版本的 secret*** *时，此解决方案才有效。如果您设置静态版本，它将不起作用，因为* ***秘密的版本*** *在秘密管理器上是不可变的。您只能添加/删除版本，而* ***伪版本***`***latest***`**始终引用最新版本。***

# **适合正确使用情形的正确功能**

**Cloud Run 提供了 Secret Manager 服务的不同集成模式。一个是**更容易使用**，并且**的好处是在实例运行时不会改变**。
另一个是**更通用，适应最新的秘密版本**。**

**但是，请记住，在生产环境中使用 `**latest**` **版本的**机密**必须被**很好地记录和假定**。
使用已定义的版本**允许您一致地部署和回滚**到某个时间点。****

**没有完美的解决方案，但是**你有所有的选择**！！选择正确的一个，并**建立可怕的，安全的东西！！****

# **自己试试吧**

**如果你想试一试，有几个步骤**

**创造一个秘密**

```
**echo -n "hello" | gcloud secrets create medium --data-file=-** 
```

**授予云运行默认服务帐户*(如果您为您的部署使用默认服务帐户，即使客户管理的服务帐户是首选，它也足以用于测试)***

```
**gcloud secrets add-iam-policy-binding medium \
 --member=serviceAccount:<PROJECT NUMBER>-compute@developer.gserviceaccount.com \
 --role=roles/secretmanager.secretAccessor**
```

***用自己的项目号*替换 `*PROJECT NUMBER*`**

***然后，您可以在 Go 中使用这段代码***

```
***package main

import (
 "fmt"
 "io/ioutil"
 "net/http"
 "os"
)

func main() {
 http.HandleFunc("/", readSecret)
 http.ListenAndServe(":8080", nil)
}

func readSecret(w http.ResponseWriter, r *http.Request) {
 secFile, _ := ioutil.ReadFile("/secret/mysecret")
 secEnvVar := os.Getenv("mysecret")
 fmt.Fprintf(w, "the file secret value is %s.\nThe env var secret value is %s", string(secFile), secEnvVar)
}***
```

***并在云上部署它***

```
***gcloud run deploy secret-read \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --source=. \
  --set-secrets=/secret/mysecret=medium:latest,mysecret=medium:latest***
```

***该命令构建容器并同时部署它。请注意，该服务不是私有的(所有用户都可以访问它)，您必须使用最新版本的秘密才能使其工作。***

***点击提供的链接，您可以看到文件和环境变量有相同的值。***

***现在，给你的秘密添加一个新版本。***

```
***echo -n "bye" | gcloud secrets versions add medium --data-file=-***
```

***并重新加载云运行服务页面。
*你必须在前一页载入后的 15 分钟内完成。如果花费更多的时间，实例将被卸载，并使用环境变量中的新秘密负载运行一个新实例。****

**这一次，您可以看到只有**秘密通读文件具有新的秘密值**，而不是环境变量。**