# 如何在顶点人工智能管道中使用之前创建的工件

> 原文：<https://medium.com/google-cloud/how-to-use-previously-created-artifacts-with-vertex-ai-pipelines-ca440bb3f401?source=collection_archive---------1----------------------->

谷歌为 Vertex AI 管道提供了一堆[预建组件](https://cloud.google.com/vertex-ai/docs/pipelines/gcpc-list)。这些组件产生工件作为输出，并将它们作为其他预构建组件的输入。

**只要那些工件是在同一个管道内生产和消费的，我们就可以很容易地使用它们(在这种情况下，就没有必要继续阅读这篇文章)。**

但是在某些情况下，我们需要重用由另一个管道或管道运行产生的工件。这些艺术品也可以完全在任何…