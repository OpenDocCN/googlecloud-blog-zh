# 技巧#18:从 GCS 加载 JSON 数据并在工作流中使用

> 原文：<https://medium.com/google-cloud/tip-18-load-and-use-json-data-in-your-workflow-from-gcs-898d8e5927f?source=collection_archive---------3----------------------->

跟进关于在云存储桶中写入和读取 JSON 文件的[文章](/google-cloud/day-16-with-workflows-reading-in-and-writing-a-json-file-to-a-storage-bucket-from-a-workflow-48b85c12d225)，我们看到我们可以访问 JSON 文件的数据，并在我们的工作流中使用它。让我们来看看这个的具体用法。

今天，我们将利用这种机制来避免对我们从工作流中调用的 API 的 URL 进行硬编码。这样，它使得工作流在不同的环境中更容易移植。

让我们将读取和加载 JSON 数据的逻辑重组到一个可重用的子工作流中:

```
read_env_from_gcs:
    params: [bucket, object]
    steps:
    - read_from_gcs:
        call: http.get
        args:
            url: ${"[https://storage.googleapis.com/download/storage/v1/b/](https://storage.googleapis.com/download/storage/v1/b/)" + bucket + "/o/" + object}
            auth:
                type: OAuth2
            query:
                alt: media
        result: env_file_json_content
    - return_content:
        return: ${env_file_json_content.body}
```

您用两个参数调用这个子工作流:bucket 名称，以及您想要加载的对象或文件名。

现在让我们从主工作流中使用它。我们需要第一步调用子工作流，从特定的桶中加载特定的文件。下面的子工作流将返回 env_details 变量中 JSON 数据的内容。

```
​​main:
    params: [input]
    steps:
    - load_env_details:
        call: read_env_from_gcs
        args:
            bucket: workflow_environment_info
            object: env-info.json
        result: env_details
```

假设 JSON 文件包含一个带有 SERVICE_URL 键的 JSON 对象，指向一个服务的 URL，那么你可以用下面的表达式调用这个服务:${env_details。SERVICE_URL}如下所示。

```
- call_service:
        call: http.get
        args:
            url: ${env_details.SERVICE_URL}
        result: service_result
    - return_result:
        return: ${service_result.body}
```

这对于避免在工作流定义中硬编码某些值非常有用。然而，对于真正的特定于环境的部署，这还不理想，因为您必须指向 bucket 中的不同文件，或者使用不同的 bucket。并且当您调用子工作流时，该信息当前被硬编码在定义中。但是如果您遵循一些映射到环境的项目名称和存储桶名称的命名约定，这是可行的！(即。PROD_bucket vs DEV_bucket，或者 PROD-env-info . JSON vs DEV-env-info . JSON)

让我们等待工作流中环境变量的支持！

*最初发表于*[T5【https://glaforge.appspot.com】](https://glaforge.appspot.com/article/load-and-use-json-data-in-your-workflow-from-gcs)*。*