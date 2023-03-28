# Apache Beam 的错误处理:Asgarde 演示

> 原文：<https://medium.com/google-cloud/error-handling-with-apache-beam-presentation-of-asgarde-ce5e5dfb2d74?source=collection_archive---------0----------------------->

![](img/4829188ea3caceaed2ce4b4d660ee564.png)

# 波束 ParDo 和 DoFn 的误差逻辑

Beam 建议用死信来处理错误。这意味着捕捉流程中的错误，并使用辅助输出，将错误收集到文件、数据库或任何其他输出中…

Beam 建议用`DoFn`类中的`TupleTags`处理侧输出，例如:

使用这种方法，我们可以在所有步骤中获得输出和失败结果集合。

# 波束映射元素和平面映射元素的错误逻辑

Beam 还允许通过内置组件处理错误，如`MapElements`和`FlatMapElements`(目前这是一个截至 2020 年 4 月的实验性功能)。

在幕后，在这些类中，Beam 使用了上面解释的相同概念。

示例:

对于 FlatMapElements，逻辑是相同的:

# 方法之间的比较

# 普通光束管道

在通常的束流水线流程中，步骤被流畅地链接:

# 具有错误处理的普通波束流水线

以下是每个步骤中错误处理的相同流程:

这种方法的问题:

*   我们在应用链上失去了原有的流畅风格，因为我们必须处理每一步的输出和错误。
*   对于`MapElements`和`FlatMapElements`我们总是要加上`exceptionsInto`和`exceptionsVia`(可以集中)。
*   对于每个定制的 DoFn，我们必须复制`TupleTag`逻辑和 try catch 块的代码(可以是集中式的)。
*   代码很冗长。
*   没有集中的代码把所有的错误串联起来，我们要把所有的失败串联起来(可以集中)。

我们开发了一个名为`Asgarde`的库，它允许在 Beam 管道中进行错误处理，代码不那么冗长，表达性代码更加简洁&。

# 使用 Asgarde 进行错误处理的普通射束管道

下面是错误处理的相同流程，但是使用了这个库:

一些解释:

*   `CollectionComposer`类允许集中所有的错误逻辑，流畅地组合应用并连接流中发生的所有故障。
*   对于`MapElements`和`FlatMapElements`，在场景后面，`apply`方法在`Failure`对象上添加`exceptionsInto`和`exceptionsVia`。如果需要，如果你有一些基于`Failure`对象的自定义逻辑，我们也可以显式地使用`exceptionsInto`和`exceptionsVia`。
*   `MapElementFn`类是一个定制的`DoFn`类，在内部封装了`DoFn`的共享逻辑，比如 try/catch 块和 Tuple 标签。我们将在下一节中详细介绍概念。

# 图书馆的目的

*   将所有错误处理逻辑包装在一个 composer 类中。
*   将`exceptionsInto`和`exceptionsVia`用在原生光束类`MapElements`和`FlatMapElements`中。
*   在检查故障时，保持 Beam 在`apply`方法中提出的流畅风格，并提供一种不太冗长的错误处理方式。
*   用集中的 try/catch 块(loan 模式)和 Tuple 标签公开定制的`DoFn`类。
*   公开对`DoFn`类的`@Setup`步骤的更容易的访问。
*   揭示一种处理过滤逻辑中的错误的方法(目前 Beam 的`Filter.by`不提供)。

贷款模式的一些资源:

[https://dzone . com/articles/functional-programming-patterns-with-Java-8](https://dzone.com/articles/functional-programming-patterns-with-java-8)

[](https://blog.knoldus.com/scalaknol-understanding-loan-pattern/) [## ScalaKnol:了解贷款模式

### 顾名思义，借出模式将资源借出给你的函数。所以如果你把这个句子。它会…

blog.knoldus.com](https://blog.knoldus.com/scalaknol-understanding-loan-pattern/) 

关于`Asgarde`库的更多信息，这里是项目自述文件的链接:

[](https://github.com/tosun-si/asgarde) [## GitHub-tosun-si/as garde:as garde 允许用 Apache Beam Java 简化错误处理，用…

### 这个模块允许用 Apache Beam Java 简化错误处理。该项目托管在 Maven 存储库上。你可以…

github.com](https://github.com/tosun-si/asgarde) 

> 如果你喜欢我的文章，想看我的帖子，请关注我:
> 
> - [中](/@mazlum.tosun)
> - [推特](https://twitter.com/MazlumTosun3)
> - [领英](https://www.linkedin.com/in/mazlum-tosun-900b1812)