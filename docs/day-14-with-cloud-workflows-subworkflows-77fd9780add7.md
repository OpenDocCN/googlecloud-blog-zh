# 云工作流的第 14 天:子工作流

> 原文：<https://medium.com/google-cloud/day-14-with-cloud-workflows-subworkflows-77fd9780add7?source=collection_archive---------1----------------------->

工作流由一系列步骤和分支组成。有时，一些特定的步骤序列可以重复，在工作流定义中避免容易出错的重复是一个好主意(特别是如果您在一个地方进行了更改，而忘记在另一个地方进行更改)。您可以通过创建子工作流来模块化您的定义，有点像编程语言中的子例程或函数。例如，昨天，我们看了一下[如何登录云日志](http://glaforge.appspot.com/article/day-13-with-cloud-workflows-logging-with-cloud-logging):如果你想在工作流的几个地方登录，你可以在子工作流中提取那个例程。

让我们在下面的视频中看到这一点，之后你可以阅读所有的解释:

首先，让我们回过头来看看工作流定义的结构。您直接在主 YAML 文件中编写一系列步骤。借助，您可以在步骤之间来回移动，但是使用跳转来模拟子例程并不方便(还记得 BASIC 及其 gotos 的辉煌过去吗？).相反，云工作流允许你在一个“主”下分离步骤，子例程在它们自己的子例程名称下。

到目前为止，我们只有一系列步骤:

```
main:
    steps:
        - stepOne:
            ...
        - stepTwo:
            ...
        - stepThree:
           ...
```

这些步骤隐含在主例程中。下面是如何通过主程序块及其下面的步骤来明确显示这个主例程:

```
subWorkflow:
    params: [param1, param2, param3: "default value"]
    steps:
        - stepOne:
            ...
        - stepTwo:
            ...
        - stepThree:
           ...
```

为了创建子工作流，我们遵循相同的结构，但是使用不同的名称，但是您也可以像这样传递参数:

```
main:
    steps:
        - greet:
            call: greet
            args:
                greeting: "Hello"
                name: "Guillaume"
            result: concatenation
        - returning:
            return: ${concatenation}greet:
    params: [greeting, name: "World"]
    steps:
        - append:
            return: ${greeting + ", " + name + "!"}
```

请注意，您可以传递几个参数，并且当调用站点没有提供该参数时，参数可以有默认值。

然后，在您的主流程中，您可以使用 call 指令调用该子工作流。让我们来看一个具体的例子，它只是连接了两个字符串:

在 call 指令中，我们传递问候语和名称参数，结果将包含子工作流调用的输出。在子工作流中，我们定义了我们的参数，并且我们有一个单独的步骤，只需返回一个表达式，它就是所需的问候消息连接。

最后一个例子，但可能比连接字符串更有用！让我们把昨天的云日志集成变成一个可重用的子工作流。这样，您将能够在您的主工作流定义中根据需要多次调用日志子工作流，而无需重复您自己:

```
main:
  steps:
    - first_log_msg:
        call: logMessage
        args:
          msg: "First message"
    - second_log_msg:
        call: logMessage
        args:
          msg: "Second message"

logMessage:
  params: [msg]
  steps:
    - log:
        call: http.post
        args:
            url: [https://logging.googleapis.com/v2/entries:write](https://logging.googleapis.com/v2/entries:write)
            auth:
                type: OAuth2
            body:
                entries:
                    - logName: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/logs/workflow_logger"}
                      resource:
                        type: "audited_resource"
                        labels: {}
                      textPayload: ${msg}
```

瞧啊。我们在主工作流中调用了两次 logMessage 子工作流，只是传递文本消息来登录云日志。

*最初发表于*[T5【http://glaforge.appspot.com】](http://glaforge.appspot.com/article/day-14-with-cloud-workflows-subworkflows)*。*