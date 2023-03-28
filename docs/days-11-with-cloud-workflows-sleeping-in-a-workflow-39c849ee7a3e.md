# 云工作流的第 11 天:在工作流中睡觉

> 原文：<https://medium.com/google-cloud/days-11-with-cloud-workflows-sleeping-in-a-workflow-39c849ee7a3e?source=collection_archive---------0----------------------->

工作流不一定是即时的，执行可以跨越很长一段时间。有些步骤可能会启动异步操作，这可能需要几秒钟或几分钟才能完成，但在该过程结束时不会通知您。因此，当您想要完成某件事情时，例如在再次轮询以检查异步操作的状态之前，您可以在工作流中引入休眠操作。

为了引入一个[睡眠操作](https://cloud.google.com/workflows/docs/reference/syntax)，在工作流中添加一个调用内置睡眠操作的步骤:

```
- someSleep:
    call: sys.sleep
    args:
        seconds: 10
- returnOutput:
    return: We waited for 10 seconds!
```

休眠操作采用 seconds 参数，您可以在该参数中指定等待的秒数。

通过结合条件跳转和休眠操作，您可以轻松地定期轮询一些资源或 API，以双重检查它是否已完成。

*最初发表于*[T5【http://glaforge.appspot.com】](http://glaforge.appspot.com/article/days-11-with-cloud-workflows-sleeping-in-a-workflow)*。*