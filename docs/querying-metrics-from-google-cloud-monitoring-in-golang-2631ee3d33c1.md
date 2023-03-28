# 在 Golang 中查询来自 Google 云监控的指标

> 原文：<https://medium.com/google-cloud/querying-metrics-from-google-cloud-monitoring-in-golang-2631ee3d33c1?source=collection_archive---------0----------------------->

![](img/ff4e2bb196093622da704373cfbf674d.png)

卢克·切瑟在 [Unsplash](https://unsplash.com/s/photos/monitoring?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 上的照片

今天，我们需要从我们的 [Google Pub/Sub](https://cloud.google.com/pubsub) 订阅中获取未确认消息的数量，以扩展我们的接收渠道。为了做到这一点，我们简单地查询了来自[谷歌云监控](https://cloud.google.com/monitoring)的未确认消息的数量。我们需要的指标是[subscription/num _ un delivered _ messages](https://cloud.google.com/monitoring/api/metrics_gcp#pubsub/subscription/num_undelivered_messages)，它每 60 秒采样一次，需要 120 秒才可见。我们需要考虑到这一点，以便适当地扩展管道。

因此，我们在 Golang 中实现了一个简单的解决方案来查询云监控的指标，如下所示: