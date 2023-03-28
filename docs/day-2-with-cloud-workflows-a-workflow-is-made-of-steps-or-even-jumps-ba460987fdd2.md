# 云工作流的第二天:工作流由步骤甚至跳跃组成！

> 原文：<https://medium.com/google-cloud/day-2-with-cloud-workflows-a-workflow-is-made-of-steps-or-even-jumps-ba460987fdd2?source=collection_archive---------0----------------------->

让我们继续探索[云工作流](https://cloud.google.com/workflows)！

昨天，我们发现了云工作流的 UI。我们[创建了我们的第一个工作流](http://glaforge.appspot.com/article/day-1-with-cloud-workflows-your-first-step-to-hello-world)。我们从一个简单的步骤开始，返回一条问候消息:

```
- sayHello:
    return: Hello from Cloud Workflows!
```

工作流定义由步骤组成。但不只是一个！您可以创建几个步骤。在 YAML，你的工作流程的结构应该是这样的:

```
- stepOne:
    # do something
- stepTwo:
    # do something else
- sayHello:
    return: Hello from Cloud Workflows!
```

默认情况下，步骤按照出现的顺序从上到下执行。当您返回值或到达最后一步时，执行将结束。如果没有 return 语句，则作为工作流执行的结果返回一个空值。

一个工作流执行的一小步，但是你也可以在步骤之间进行跳跃！为此，您将使用下一条指令:

```
- stepOne:
    next: stepTwo
- stepThree:
    next: sayHello
- stepTwo:
    next: stepThree
- sayHello:
    return: Hello from Cloud Workflows!
```

在这里，我们在返回一个值的最后一个步骤之前，在各个步骤之间来回跳转，从而完成工作流的执行。

当然，我们可以超越一系列线性步骤，在后续文章中，我们将看到如何为更复杂的逻辑创建条件跳转和开关，以及如何在步骤之间传递一些数据和值。

【http://glaforge.appspot.com】最初发表于[](http://glaforge.appspot.com/article/day-2-with-cloud-workflows-a-workflow-is-made-of-steps-or-even-jumps)**。**