# 工作流程提示和技巧

> 原文：<https://medium.com/google-cloud/workflows-tipsn-tricks-d7196eb5098d?source=collection_archive---------0----------------------->

以下是我们在使用[工作流](https://cloud.google.com/workflows)时发现的一些有用的一般提示和技巧:

**避免硬编码网址**

因为工作流是关于调用 API 和服务 URL 的，所以有一些干净的方法来处理这些 URL 是很重要的。您可以将它们硬编码到您的工作流定义中，但是问题是您的工作流会变得更难维护。特别是，当您在多个环境中工作时会发生什么？您必须复制您的 YAML 定义，并为生产环境、暂存环境和开发环境使用不同的 URL。在多个文件中对本质上相同的工作流进行修改容易出错，并且很快变得很痛苦。为了避免对这些 URL 进行硬编码，有几种方法。

第一个是将这些 URL 具体化，[将它们作为工作流执行参数](https://github.com/GoogleCloudPlatform/workflows-demos/tree/master/multi-env-deployment#option-1-use-urls-as-runtime-arguments)传递。这对于通过 CLI、通过各种客户端库或其余的&gRPC API 启动的工作流执行非常有用。然而，在事件触发的工作流的情况下，第一种方法有一个限制，其中调用者是 Eventarc。在这种情况下，Eventarc 决定传递哪些参数(即事件有效载荷)。在这种情况下，无法传递额外的参数。

更安全的方法是使用一些占位符替换技术。在部署更新的定义之前，只需使用一个工具来替换定义文件中的一些特定字符串标记。我们使用一些 Clo [ud 构建步骤探索了这种方法，这些步骤执行一些字符串替换](https://github.com/GoogleCloudPlatform/workflows-demos/tree/master/multi-env-deployment#option-2-use-cloud-build-to-deploy-multiple-versions)。您仍然有一个单一的工作流定义文件，但是您为不同的环境部署了变体。如果您正在使用 Terraform 来配置您的基础设施，我们会为您提供帮助，您也可以使用类似于 Terraform 的[技术。](https://github.com/GoogleCloudPlatform/workflows-demos/tree/master/multi-env-deployment#option-3-use-terraform-to-deploy-multiple-versions)

还有其他可能的方法，比如利用 Secret Manager 和专用的[工作流连接器](https://cloud.google.com/workflows/docs/reference/googleapis/secretmanager/Overview)，来存储和检索这些 URL。或者你也可以[在一个云存储桶](https://github.com/GoogleCloudPlatform/workflows-demos/tree/master/gcs-read-write-json#load-environment-specific-variables-from-a-json-file-in-gcs)中读取一些 JSON 文件，在其中你将存储那些环境特定的细节。

**利用子步骤**

除了分支或循环，定义步骤是一个相当连续的过程。一步接一步。步骤是它们自己的原子操作。然而，通常情况下，一些步骤实际上是齐头并进的，比如进行 API 调用、记录其结果、检索部分有效负载并将其分配到一些变量中。实际上，您可以将普通步骤重组为子步骤。当您从一组步骤转移到另一组步骤时，这变得很方便，而不必指向正确的原子步骤。

```
main:
    params: [input]
    steps:
    - callWikipedia:
        steps:
        - checkSearchTermInInput:
            switch:
                - condition: ${"searchTerm" in input}
                  assign:
                    - searchTerm: ${input.searchTerm}
                  next: readWikipedia
        - getCurrentTime:
            call: http.get
            args:
                url: …
            result: currentDateTime
        - setFromCallResult:
            assign:
                - searchTerm: ${currentDateTime.body.dayOfTheWeek}
        - readWikipedia:
            call: http.get
            args:
                url: [https://en.wikipedia.org/w/api.php](https://en.wikipedia.org/w/api.php)
                query:
                    action: opensearch
                    search: ${searchTerm}
            result: wikiResult
    - returnOutput:
            return: ${wikiResult.body[1]}
```

**换行表达式**

美元/花括号＄{ }表达式不是 YAML 规范的一部分，所以你放进去的东西有时与 YAML 的期望不太相符。例如，在表达式的字符串中放一个冒号可能会有问题，因为 YAML 解析器认为冒号是 YAML 键的结尾，是右边的开始。所以为了安全起见，你可以用引号将表达式括起来，比如:' ${…} '

表达式可以跨越几行，也可以跨越表达式中的字符串。这对于针对大查询的 SQL 查询来说很方便，就像我们的[示例](https://github.com/GoogleCloudPlatform/workflows-demos/tree/master/bigquery-parallel)中一样:

```
query: ${
    "SELECT TITLE, SUM(views)
    FROM `bigquery-samples.wikipedia_pageviews." + table + "`
    WHERE LENGTH(TITLE) > 10
    GROUP BY TITLE
    ORDER BY SUM(VIEWS) DESC
    LIMIT 100"
}
```

**用声明式 API 调用替换无逻辑服务**

在我们的[无服务器车间](https://github.com/GoogleCloudPlatform/serverless-photosharing-workshop/)的实验室 1 中，我们有一个[功能服务](https://github.com/GoogleCloudPlatform/serverless-photosharing-workshop/blob/master/functions/image-analysis/nodejs/index.js#L19)，它调用云视觉 API，检查布尔属性，然后将结果写入 Firestore。但是可以从工作流中以声明方式调用 Vision API。布尔检查可以用一个开关条件表达式来完成，甚至写入 Firestore 也可以通过一个声明性 API 调用来完成。当在[实验室 6](https://github.com/GoogleCloudPlatform/serverless-photosharing-workshop/blob/master/workflows/workflows.yaml#L33) 重写我们的应用程序以使用编排的方法时，我们将那些无逻辑的调用转移到声明性 API 调用中。

有时工作流缺少一些您需要的内置函数，因此您别无选择，只能使用一个函数来完成工作。但是当你有一些非常没有逻辑的代码，只是进行一些 API 调用时，你最好使用 Workflows 语法来声明性地编写它。

这并不意味着所有的事情，或者尽可能多的事情，都应该在工作流中以声明的方式完成。Workflows 不是锤子，也绝对不是编程语言。因此，当存在真正的逻辑时，您肯定需要调用一些代表该业务逻辑的服务。

**储存你需要的，释放你能释放的**

工作流不断地向工作流执行授予更多的内存，但是有时候，对于较大的 API 响应负载，您会很高兴有更多的内存。这时，明智的内存管理可能是一件好事。你可以有选择地存储变量:不要存储太多，只存储你真正需要的有效载荷部分。一旦你知道你不需要某个变量的内容，你也可以把这个变量重新赋值为 null，这样也可以释放内存。同样，首先，如果 API 允许您更积极地过滤结果，您也应该这样做。最后但并非最不重要的一点是，如果您调用的服务返回一个巨大的负载，而这个负载在工作流内存中放不下，那么您可以将调用委托给自己的函数，这个函数会代表您进行调用，并只返回您真正感兴趣的部分。

不要忘记查看关于[配额和限制](https://cloud.google.com/workflows/quotas)的文档，以了解更多可能的情况。

**利用子工作流和调用外部工作流的能力**

在您的工作流程中，有时您可能需要重复一些步骤。这时[子工作流](https://cloud.google.com/workflows/docs/reference/syntax/subworkflows)就变得很方便了。子工作流类似于子例程、程序或方法。它们是一种使一组步骤可在工作流的几个地方重用的方法，可能用不同的参数进行参数化。唯一的缺点可能是子工作流只是工作流定义的一部分，所以它们不能在其他工作流中重用。在这种情况下，您实际上可以创建一个专用的可重用工作流，因为您也可以[从其他工作流中调用工作流](https://cloud.google.com/workflows/docs/reference/googleapis/workflowexecutions/Overview)！用于工作流的工作流连接器可以提供帮助。

**总结**

我们已经介绍了一些提示和技巧，并且回顾了一些关于如何充分利用工作流的有用建议。当然还有其他我们忘记了的。所以请在 Twitter 上与 [@meteatamel](https://twitter.com/meteatamel) 和 [@glaforge](https://twitter.com/glaforge) 分享它们。

不要忘记仔细检查工作流程[文档](https://cloud.google.com/workflows/docs)中的内容。特别是，看看[标准库](https://cloud.google.com/workflows/docs/reference/stdlib/overview)的内置函数，看看[您可以使用的连接器](https://cloud.google.com/workflows/docs/reference/googleapis)列表，甚至可能打印出[语法备忘单](https://cloud.google.com/workflows/docs/reference/syntax/syntax-cheat-sheet)！

最后，在文档门户中查看所有的[示例](https://cloud.google.com/workflows/docs/samples)，以及所有的[工作流演示](https://github.com/GoogleCloudPlatform/workflows-demos) Mete 和我已经构建并开源了。