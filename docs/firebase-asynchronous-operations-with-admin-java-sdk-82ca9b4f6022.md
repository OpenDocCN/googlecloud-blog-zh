# Firebase:使用管理 Java SDK 的异步操作

> 原文：<https://medium.com/google-cloud/firebase-asynchronous-operations-with-admin-java-sdk-82ca9b4f6022?source=collection_archive---------0----------------------->

![](img/a183d23ca190768be82fdd3909d89379.png)

[Firebase Admin Java SDK](https://firebase.google.com/docs/admin/setup) 的版本 [5.4.0](https://firebase.google.com/support/release-notes/admin/java#5.4.0) 对 SDK 处理线程和异步操作的方式进行了重大改进。在这些变化中，`[Task](https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/tasks/Task)`界面的弃用和`[ApiFuture](https://googleapis.github.io/api-common-java/1.1.0/apidocs/com/google/api/core/ApiFuture.html)`界面的引入可能是最值得注意的。这些变化跨越了 Admin Java SDK 公开的多个 API，并对开发人员如何使用 SDK 编写代码产生了显著的影响。这篇文章讨论了这些变化背后的基本原理，并解释了如何将 Java 应用程序从 Tasks 迁移到 ApiFutures。

Admin Java SDK 中的大多数公共 API 方法都是异步的。也就是说，调用方不会在方法上受阻。SDK 只是将方法体的执行提交给一个工作线程池，然后立即返回给调用者一个表示提交操作的对象。直到 SDK 的 5.4.0 版本，这个返回的对象是一个`Task<T>`的实例，其中通用类型参数`T`表示异步操作产生的结果的类型。例如，[创建定制 jwt](https://firebase.google.com/docs/auth/admin/create-custom-tokens)的`FirebaseAuth.createCustomToken()`方法将返回一个`Task<String>`。

`Task`接口支持注册在底层异步操作完成时触发的回调。它还有助于延续——即链接多个异步操作，这样一个操作产生的结果可以通过管道传递给另一个操作。

Admin Java SDK 的 5.4.0 版本不赞成使用`Task`接口，以及所有返回`Task<T>`的公共 API 方法。SDK 现在推荐使用新引入的返回`ApiFuture<T>`的`*Async()`方法。例如，代替`FirebaseAuth.createCustomToken()`，开发人员现在被建议使用`FirebaseAuth.createCustomTokenAsync()`方法，它返回一个`ApiFuture<String>`。SDK 中的所有其他公共方法都引入了类似的替换。这些新方法在功能上等同于它们不推荐使用的对应方法。也就是说，它们还将方法执行提交给一个工作线程池，并立即返回。但是表示异步操作的返回对象属于不同的类型。

在架构上，`Task`和`ApiFuture`接口代表了两种不同风格的异步编程。因此，他们看起来很不一样。然而，它们可以用来实现相同的用例，因此从一个用例迁移到另一个用例通常是微不足道的，我们很快就会看到。

## 为什么不推荐`Task interface`？

此举有两个主要原因:

1.  大多数服务器端 Java 库——包括[谷歌云平台(GCP)](https://cloud.google.com/) 提供的库——使用 Java 的内置`[Future](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html)`接口(或其子接口)公开异步操作。我们希望 Admin Java SDK 看起来更像那些库，这样服务器端的 Java 开发人员会有宾至如归的感觉。我们还想在同一个应用程序中混合使用 Firebase 和 GCP 库时，提供一致的开发人员体验。
2.  在 Admin Java SDK 的早期，`Task`接口和相关的实用程序是从[Android Google Mobile Services(GMS)API](https://developers.google.com/android/reference/com/google/android/gms/tasks/Task)中派生出来的。将这些代码与 Android GMS 保持同步，并且维护它们变得很麻烦。

作为 Android API 的界面应该不会让你感到惊讶。这个 API 提倡的*基于回调的*编程风格在 Android 这样的客户端平台上很流行。客户端应用程序通常依赖于图形用户界面(GUI)。因此，他们被期望具有高度的交互性和响应性。想象一个有按钮的移动应用程序，点击按钮就可以进行一些计算，结果显示在屏幕上。`Task`是实现这个用例的完美抽象。当点击按钮时，应用程序可以启动一个`Task`，GUI 更新逻辑可以作为对那个`Task`的回调来实现。这样，接收用户输入的 GUI 工作线程永远不会被阻塞，GUI 始终保持响应。

相比之下，大多数服务器端应用程序不必是交互式的。它们通常没有 GUI，而是为大规模的机器对机器交互而设计的。因此，服务器端 Java 应用程序倾向于优先考虑规模和减少端到端请求延迟，而不是 GUI 体验。在这种情况下，开发人员通常需要的是简单的 [*fork-join*](https://en.wikipedia.org/wiki/Fork%E2%80%93join_model) 风格的异步编程。这是一个程序在并行线程中分叉一组潜在的昂贵操作的地方，如果需要的话，以后再加入它们。这个`Future`界面非常棒。

考虑到这些复杂性，很明显为什么基于 Futures 的 API 更适合管理 Java SDK。然而，我们也不想完全放弃回调。在许多事件驱动和反应式系统中，回调是必须的(即使在服务器端)。因此，我们没有选择内置的`Future`接口，而是从 [Google API Common](https://github.com/googleapis/api-common-java) 项目中选择了一个更丰富的子接口——T6。ApiFutures 提供了两个世界的最佳选择。它们源自 Java 的同一个内置`Future`接口，但也支持添加回调。此外，几个 GCP 库也采用了 ApiFutures，这就更有理由在 Firebase Admin Java SDK 中使用相同的接口。

## 从任务迁移到 ApiFutures

下面是一组代码示例，演示了如何将 Java 代码从任务迁移到 ApiFutures。每个示例展示了任务的某个特性，以及使用 ApiFutures 的等价代码。如果您有任何使用任务的代码，那么您很可能会大量使用回调。因此，让我们从演示如何向`ApiFuture`添加回调开始。

清单 1:向任务和 ApiFuture 添加回调

这个例子有几个值得注意的地方:

1.  任务接口直接公开添加回调的方法(如`addOnSuccessListener()`和`addOnFailureListener()`)。但是要给`ApiFuture`添加回调，必须使用`[ApiFutures](https://googleapis.github.io/api-common-java/1.1.0/apidocs/com/google/api/core/ApiFutures.html)`助手类中的静态`addCallback()`方法。
2.  `Task`接口便于添加单独的回调来处理成功、失败和完成事件。另一方面,`ApiFuture`有一个不太灵活的 API 来添加回调，从某种意义上说，它只支持一种类型的回调——`[ApiFutureCallback](https://googleapis.github.io/api-common-java/1.1.0/apidocs/com/google/api/core/ApiFutureCallback.html)`,它处理成功和失败事件。
3.  在这两个 API 接口中，对于可以添加的回调的数量没有限制，并且对于回调执行时的顺序也没有保证。

好奇的开发人员可能想知道清单 1 中添加的`ApiFutureCallback`在哪一个线程上执行。如果在调用`addCallback()`时底层异步操作还没有完成，回调将在异步操作完成时在同一线程上运行。如果操作已经完成，回调会立即在调用`addCallback()`的线程上运行。当回调简单而短暂时，这种行为已经足够好了。如果你的回调是昂贵的，或者如果你希望在这方面有更多可预测的语义，你应该使用接受一个`[Executor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html)`作为第三个参数的`ApiFutures.addCallback()`覆盖。 *[* ***注意:*** *此覆盖仅在 Google API Common 的 1.2.0 版本之后可用，而最新的 Admin SDK (5.5.0)随 1.1.0 版本一起提供。在这个问题解决之前，必须手动升级 Google API Common 才能使用新方法。】*

接下来我们将讨论如何迁移`Task`延续。continuation 转换异步操作的输出，并将聚合计算(异步操作+转换)公开为新的`Task`。这个动作的递归特性有效地实现了异步操作序列的链接。清单 2 显示了将由`Task<String>`产生的字符串转换成字符串列表的延续。产生的聚合计算由`Task<List<String>>`的实例表示。

清单 2:任务和 ApiFutures 的延续

在这个例子中，`[ApiFunction](https://googleapis.github.io/api-common-java/1.1.0/apidocs/com/google/api/core/ApiFunction.html)`接口扮演了现在已经废弃的`[Continuation](https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/tasks/Continuation)`接口的角色。`ApiFutures.transform()`助手方法将产生的聚合计算作为`ApiFuture<List<String>>`的实例公开。还有一个`[ApiAsyncFunction](https://googleapis.github.io/api-common-java/1.1.0/apidocs/com/google/api/core/ApiAsyncFunction.html)`接口和一个`ApiFutures.transformAsync()`助手方法，用于转换本身需要进行异步调用的情况。

清单 3 演示了如何在 ApiFutures 中包装值和异常。使用 Admin SDK 的开发人员几乎没有理由使用它，因为 SDK 已经提供了返回 ApiFutures 的方法。这里提到它是为了完整性。

清单 3:包装值和异常

在向立即`ApiFuture`添加回调或转换时要小心。它们将总是在`addCallback()`或`transform()`方法的调用线程上执行。如果这不是所期望的行为，那么在调用那些帮助器方法时应该指定一个显式的`Executor`。

清单 4 展示了如何等待异步操作完成。`Tasks`助手类需要等待一个`Task`。但是对于一个`ApiFuture`来说就相当琐碎了。

清单 4:等待任务和 ApiFutures

有时我们想创建一个不完整的异步操作，稍后再完成它。当您希望将任意代码块表示为异步操作时，这通常很有用。对于任务，这是通过使用`[TaskCompletionSource](https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/tasks/TaskCompletionSource)`实现的。清单 5 中的最后一个代码示例展示了如何用一个`ApiFuture`替换一个`TaskCompletionSource`。

清单 5:不完整的异步操作

## 结论

自从 Java Admin SDK 的 5.4.0 版本发布以来，我已经看到了一些关于`ApiFuture`接口的更多细节和例子的询问。我希望这篇文章能解决这些问题，同时缓解从 T1 到 T2 移民的紧张局势。我还邀请 Firebase Java 社区来看看新添加的`[ThreadManager](https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/ThreadManager)`接口，它支持为 Admin SDK 配置线程池和线程工厂。它对 SDK 如何调度和执行异步操作提供了更好的控制。

展望未来，我们可以期待 Admin SDK 中围绕`ApiFuture`支持的更多特性、稳定性和文档。Firebase 还使管理 SDK APIs 与各种 GCP 库的无缝混合和匹配变得更加容易(已经支持[谷歌云存储](https://cloud.google.com/storage/)和[谷歌云 Firestore](http://firebase.google.com/docs/firestore/) )。如果你正在关注 [Java Admin SDK GitHub repo](https://github.com/firebase/firebase-admin-java/issues/55) ，你可能已经看到了从 SDK 中公开一组阻塞 API 的努力。在评论和 GitHub 上分享你的想法。让我们知道您如何使用 Java Admin SDK，以及如何进一步改善开发人员的体验。