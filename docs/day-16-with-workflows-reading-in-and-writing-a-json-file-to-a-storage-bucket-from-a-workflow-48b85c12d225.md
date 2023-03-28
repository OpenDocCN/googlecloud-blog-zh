# 工作流的第 16 天:从工作流中读入 JSON 文件并将其写入存储桶

> 原文：<https://medium.com/google-cloud/day-16-with-workflows-reading-in-and-writing-a-json-file-to-a-storage-bucket-from-a-workflow-48b85c12d225?source=collection_archive---------3----------------------->

[Workflows](https://cloud.google.com/workflows) 提供了几个[连接器](https://cloud.google.com/workflows/docs/reference/googleapis)，用于与各种 Google Cloud APIs 和服务进行交互。过去，我曾使用过[文档人工智能连接器](https://cloud.google.com/workflows/docs/reference/googleapis/documentai/Overview)来解析费用收据之类的文档，或者使用[秘密管理器连接器](https://cloud.google.com/workflows/docs/reference/googleapis/secretmanager/Overview)来存储和访问密码之类的秘密。我今天感兴趣使用的另一个有用的连接器是 [Google 云存储连接器](https://cloud.google.com/workflows/docs/reference/googleapis/storage/Overview)，用于存储和读取存储在存储桶中的文件。

这些连接器是从它们的 API 发现描述符自动生成的，但是目前有一些限制，例如，阻止下载文件的内容。所以我没有使用连接器，而是查看了云存储的 JSON API，看看它提供了什么( [insert](https://cloud.google.com/workflows/docs/reference/googleapis/storage/v1/objects/insert) 和 [get](https://cloud.google.com/storage/docs/json_api/v1/objects/get) 方法)。

我想做的是存储一个 JSON 文档，并读取一个 JSON 文档。我还没有尝试过其他媒体类型，比如图片或其他二进制文件。总之，下面是如何将 JSON 文件写入云存储桶:

```
main:
    params: [input]
    steps:
    - assignment:
        assign:
            - bucket: YOUR_BUCKET_NAME_HERE
    - write_to_gcs:
        call: http.post
        args:
            url: ${"[https://storage.googleapis.com/upload/storage/v1/b/](https://storage.googleapis.com/upload/storage/v1/b/)" + bucket + "/o"}
            auth:
                type: OAuth2
            query:
                name: THE_FILE_NAME_HERE
            body:
                name: Guillaume
                age: 99
```

在这个文件中，我存储了一个 JSON 文档，其中包含几个键，这些键是在调用体中定义的。默认情况下，这里假设一个 JSON 媒体类型，所以在 YAML 底部定义的主体实际上在结果文件中被写成 JSON。哦，当然，不要忘记改变上面例子中的桶和对象的名称。

现在，您可以这样从桶中读取文件的内容:

```
main:
    params: [input]
    steps:
    - assignment:
        assign:
            - bucket: YOUR_BUCKET_NAME_HERE
            - name: THE_FILE_NAME_HERE
    - read_from_gcs:
        call: http.get
        args:
            url: ${"[https://storage.googleapis.com/download/storage/v1/b/](https://storage.googleapis.com/download/storage/v1/b/)" + bucket + "/o/" + name}
            auth:
                type: OAuth2
            query:
                alt: media
        result: data_json_content
    - return_content:
        return: ${data_json_content.body}
```

这一次我们将 GCS URL 从“upload”改为“download ”,我们使用 alt=media 查询参数来指示 GCS JSON API 我们想要检索文件的内容(不仅仅是它的元数据)。最后，我们返回包含内容的调用体。

*最初发表于*[*【https://glaforge.appspot.com】*](https://glaforge.appspot.com/article/reading-in-and-writing-a-json-file-to-a-storage-bucket-from-a-workflow)*。*