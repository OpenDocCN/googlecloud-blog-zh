# 如何获取 Java 云函数中的项目 ID？

> 原文：<https://medium.com/google-cloud/how-to-get-the-project-id-in-a-java-cloud-function-ffa0eab32ddb?source=collection_archive---------1----------------------->

当我和我的同事 Sara Ford 为即将到来的“第二代”产品测试云函数运行时时，基于 T2 云运行平台，我为 Java 运行时写了一些简单的函数。在其中一个 Java 函数中，我想使用 Google 云存储，从一个桶中下载一个文件。我看了看现有的[样本](https://github.com/googleapis/google-cloud-java/blob/main/google-cloud-examples/src/main/java/com/google/cloud/examples/storage/objects/DownloadObject.java)下载了一个对象:

```
Storage storage = StorageOptions.newBuilder()
    .setProjectId(projectId)
    .build()
    .getService();Blob blob = storage.get(BlobId.of(bucketName, objectName));
blob.downloadTo(Paths.get(destFilePath));
```

我知道 bucket 的名称，文件的名称，我将把文件存储在本地文件系统中。所以我有了所有需要的信息…除了我在其中部署 Java cloud 函数的项目 ID。那么，在云函数环境中，我如何用 Java 获得项目 ID 呢？

云函数的前一次迭代有各种有用的环境变量，包括项目 ID。所以您可以通过 System.getenv()调用来检索 ID。然而，由于各种运行时之间的各种兼容性原因，随着开源项目的发展，这个变量消失了。

然而，我知道项目 ID 也是内部[计算元数据](https://cloud.google.com/appengine/docs/standard/java/accessing-instance-metadata)的一部分，可以通过一个特殊的 URL 访问:

[http://metadata . Google . internal/computeMetadata/v1/project/project-id](http://metadata.google.internal/computeMetadata/v1/project/project-id)

有了这些知识，我想我可以简单地发出一个快速的 HTTP 请求来获取这些信息:

```
private String getProjectId() { 
    String projectId = null; 
    HttpURLConnection conn = null; 
    try {
        URL url = new URL("http://metadata.google.internal/computeMetadata/v1/project/project-id"); 
        conn = (HttpURLConnection)(url.openConnection());
        conn.setRequestProperty("Metadata-Flavor", "Google"); 
        projectId = new String(conn.getInputStream().readAllBytes(), 
                               StandardCharsets.UTF_8); 
        conn.disconnect(); 
    } catch (Throwable t) {} 
    return projectId;
}
```

为了使调用工作，必须设置您在上面看到的 Metadata-Flavor 头。我使用 Java 的内置 HttpURLConnection 来完成这项工作。还有其他的 HTTP 库可以使代码更简单，但是一开始，我不想带另一个 HTTP 客户端，只是为了检索一个简单的项目元信息。

我是为 Java 设计[函数框架的开发人员之一，该框架用于在 Java 中创建云函数，然而，我也使用 Node.js 编写了不少函数。在节点生态系统中，实际上有一个 NPM 模块，其职责是检索此类项目元数据。使用](https://github.com/GoogleCloudPlatform/functions-framework-java) [gcp-metadata](https://www.npmjs.com/package/gcp-metadata) 模块，您可以需要它，然后使用以下命令获取项目 ID:

```
const gcpMetadata = require('gcp-metadata');
const projectId = await gcpMetadata.project('project-id');
```

我很惊讶我不能很容易地在 Java 中找到一个等价的库。我花了一段时间才找到它，但它实际上也存在！那就是[com . Google . cloud:Google-cloud-core](https://googleapis.dev/java/google-cloud-core/latest/index.html)库！使用起来很简单:

```
import com.google.cloud.ServiceOptions;String projectId = ServiceOptions.getDefaultProjectId();
```

我的 pom.xml 中的一个额外的依赖项，一个对 ServiceOptions 的导入和一个静态方法调用，我就可以获得 GCP 项目 ID！因此，我现在可以将项目 ID 传递给我的 StorageOptions builder。

但是出于某种原因，我记得在我编写的一些其他项目中，我并不真正需要项目 ID 信息，因为我使用的库足够智能，可以从环境中推断出这样的信息。让我们从头再看一下存储选项。如果我只是省略 setProjectId()方法调用会怎么样？你瞧……事实上，它实际上是不需要的，项目 ID 是透明地推断出来的。所以我根本不需要搜索如何检索这个项目 ID！实际上，您可以进一步简化存储选项的创建，具体如下:

```
Storage storage = StorageOptions
    .getDefaultInstance()
    .getService();
```

至少，现在，我知道如何在 Java 中检索项目 ID，以防库或环境本身不提供这样的细节！

*原载于*[*https://glaforge.appspot.com*](https://glaforge.appspot.com/article/how-to-get-the-project-id-in-a-java-cloud-function)*。*