# 在谷歌应用引擎上扩展

> 原文：<https://medium.com/google-cloud/scaling-on-google-app-engine-903a2635f6a0?source=collection_archive---------1----------------------->

![](img/d1d01718a027432380bd261c05f38c4b.png)

## 在最近一个使用谷歌云平台的移动应用项目中，我们遇到了实例不能按预期扩展的问题。所以我们仔细看了看，以了解发生了什么。

## 例子

对于上下文，我们应该从定义实例开始:“…一个托管在谷歌基础设施上的虚拟机(VM)。”

在谷歌云平台上，根据你一次使用多少实例和使用多长时间来计费。获得正确的缩放比例非常重要，这样您就可以确保只在需要时使用资源，并且只为实际使用的资源付费。

## 应用引擎扩展

当我们开始使用 [App Engine](/@alistairsykes/queued-tasks-on-appengine-for-firebase-187972c844e1) 时，我们使用了基本的扩展配置，因为乍一看，这似乎完全符合我们的需求。

```
<basic-scaling>
    <max-instances>100</max-instances>
</basic-scaling>
```

我们的扩展配置。

很快我们就发现成本在急剧上升，而且当我们不打算使用它们时，我们似乎有一些实例会永久运行。

经过长时间的调查和咨询谷歌支持，他们告诉我们:

主要问题是，App Engine 会在您的模块中启动一个后台线程(/_ah/background)以进行日志记录，该后台线程会一直运行，直到 HTTP 请求超时

然后，他们建议我们使用最小空闲实例进行自动伸缩，并说我们希望在没有负载的时候缩小到没有实例。

自动缩放根本不支持后台线程。这意味着没有后台线程废话可以阻止我们的实例缩小。虽然这感觉像是一个工作，但我们接受了他们的建议。

## 基本与自动

看来基本缩放和自动缩放本质上是相同的。不同之处在于，自动伸缩基于一些关键指标，如请求率和响应延迟。

[https://cloud . Google . com/app engine/docs/standard/Java/an-overview-of-app...](https://cloud.google.com/appengine/docs/standard/java/an-overview-of-app-engine#scaling_types_and_instance_classes)

```
<automatic-scaling>
    <min-idle-instances>0</min-idle-instances>
    <max-idle-instances>0</max-idle-instances>
</automatic-scaling>
```

使用这种配置，我们终于开始看到我们预期的可伸缩性。实例以高负载启动，并在进入空闲状态的 15 分钟内关闭。这一页提供了更多的信息，所以请看一下——你需要向下滚动一点到相关的部分:【https://cloud.google.com/appengine/pricing

## 高负载与成本

对我们来说，成本是一个重要的考虑因素。因此，我们从理论上探讨了如何在高负载情况下优化成本。

如果你希望你的高负荷尽可能快地完成，那么你应该同时开始所有的任务。这将导致 App Engine 加速运行大量实例，以尽可能快地通过您的队列。

然而，一个实例的 15 分钟固有成本意味着启动的实例越少，达到某一点的总时间就越短。因此，如果你能控制负载，那么从战略上探索限制这个负载是值得的。

完整的故事

## [编写 API——一个移动开发者的故事](/@alistairsykes/writing-an-api-a-mobile-developers-story-8f43ec4df075)

以前的帖子

## [在 Java 中设置云 SQL](/@alistairsykes/setting-up-cloud-sql-in-java-brightec-brighton-uk-91abe16345ae)

【www.brightec.co.uk】最初发表于[](https://www.brightec.co.uk/ideas/scaling-google-app-engine)**。**