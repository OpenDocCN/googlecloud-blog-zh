# Android 推送通知— GCM 和 RxJava

> 原文：<https://medium.com/google-cloud/android-push-notifications-gcm-and-rxjava-bc3356e91ea5?source=collection_archive---------0----------------------->

## 介绍

GCM 似乎是利用 RxJava 的一个很好的例子。遵循将 GCM 与 Android 集成的文档，创建了一个 [*IntentService*](http://developer.android.com/training/run-background-service/create-service.html) ，它处理获取 GCM 令牌并将其发送到第三方服务器的过程，以便发送推送通知。

这些步骤中的每一个都需要最后一个步骤，以便完全完成注册过程，从而为推送通知完全注册用户。这个过程可以被认为是一个功能流，因此非常适合 RxJava。在失败的情况下，如重试，使用 RxJava 的[错误处理操作符](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators)操作符会有很大的好处。

本文的其余部分将假设您对 Android 的 GCM 有所了解，已经阅读了文档，并且已经了解 RxJava 的基础知识。

## 入门指南

您应该已经从 Google 获得了 API 密钥和发送者 ID。如果您还没有这样做，那么在继续下一步之前先这样做。

## 创建 IntentService

首先；创建 GCM 注册 IntentService 类，并在清单中注册它。奇怪的是，我把我的称为 GcmRegistrationIntentService。

```
<service
    android:name=".GcmRegistrationIntentService"
    android:enabled="true" />
```

请确保您覆盖了 onHandleIntent 上的*，因为这是我们将设置所需资源并通过 RxJava 启动注册流程的地方。*

我有一个*PushNotificationManager*类来包装我存储在 *SharedPreferences* 中的状态，并在应用程序收到它时保存 GCM 令牌。

我有一个 *GcmRequestManager* 类。这个类的目的是协调与我发送 GCM 令牌的 API 的网络通信。这个用的是*改型 2 (Beta)* ，已经集成了 RxJava，可以返回*可观测量*。我不打算详细说明这是如何工作的，但是我要说我返回了一个*可观察值*，任何抛出的*异常*都将以与其他示例相同的方式处理。

最后，我引用了与 GCM 相关的类 *InstanceID* 。这个类只是这个过程的一个需求，因为它是从 Google 获取 GCM 令牌的过程的抽象。

## 从 GCM 方法创建可观测量

GCM 提供的类不使用 RxJava，因此我们需要以这样一种方式包装它们，当它们的任务完成时，它们将发出一个项目。

上面的代码检索 GCM 令牌。下面您可以看到将它包装在 Observable.defer()中如何让我们创建一个 Observable，在方法完成时发出一个字符串(令牌)。

方法 *getToken()* 可以导致 *IOException* 。因此，我们可以在异常发生时捕捉它，然后发出一个内部有错误的*可观察信号*。RxJava 的一个好处是，我们可以处理下一步将要设置的*订户*的 *onError()* 中的所有异常。

类似地，当订阅*主题*时，我们可以遵循类似的过程。

成为

> 但它为什么会发出空洞的可观测信号呢？

这将是该过程的最后一步，我们不会在 *OnNext()* 中做任何事情，但是当调用 *onComplete()* 时，我们将认为该过程成功完成。如果出现任何错误，那么将调用 *onError()* 。

## 创建订户

我们需要创建一个*订阅者*来订阅 *getToken()。当 getToken()* 完成时，它将调用*sendregradationtoserver()*，后者将通过使用 *FlatMap()* 操作符*调用 *registerTopics()* 。*

在这一点上，你可能想要做不同的事情，这取决于最终的目标，但是在这个设置中，如果**任何**异常被抛出，那么整个过程必须再次完成。

我们可以使用额外的标志来表示流程的哪一部分已经完成，并从那里继续，这只是进一步简化的一种方式。

## 将所有这些放在一起…

最后一步是从 *getToken()* 开始将我们的可观察对象链接在一起，这是在 *onHandleIntent()中调用的。*

```
getToken().observeOn(Schedulers.*io*())
                .flatMap(this::sendRegistrationToServer)
                .subscribe(new GcmRegistrationSubscriber());
```

上面的代码将调用*sendrestrationtoserver()*，将收到的令牌传递给该方法。注意到

> *this::sendgregistrationtoserver*

等同于

> *() - >发送注册到服务器(令牌)*)

它将通过 *Schedulers.io()* 使用一个新线程来完成工作，因为我们不希望在主线程上运行这么长时间。

为了将*sendregradationtoserver()*链接到 *registerTopics()* 上，flatMap()操作符用于最后一次调用 *registerTopics()* 方法。

```
private Observable<Object> sendRegistrationToServer(String token) {

    pushNotificationManager.setGcmPushToken(token);
    return GcmRequestManager.*getInstance*(getApplicationContext())
            .register()
            .flatMap(this::registerTopics);
}
```

## 最终代码