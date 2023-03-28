# 谷歌云平台的 a 到 Z 个人选择— T —超时

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-t-timeouts-9d531f764dd7?source=collection_archive---------0----------------------->

超时一直在发生，并且发生在应用程序架构的不同层。我将主要讨论它们如何与 GCE & GAE 相关，而不是您在应用程序中编写的处理超时的代码。

## GCE 和超时

你有一个实例，它超时了，你从哪里开始呢？如果您希望直接访问互联网，这里的是一个很好的起点。这几段谈到了空闲的 TCP 连接，TCP 保持活动，防火墙配置，所以我不会在这里赘述。

我在 [G](/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-g-global-load-balancing-82e1b8550298#.ao27o2ah6) 谈到了云负载平衡服务，在那里引入了[实例组](https://cloud.google.com/compute/docs/instance-groups/)的概念。通过对该组设置 http 健康检查，可以检查该组中的实例上的服务是否正常运行。但是要小心，因为直接针对实例组设置的检查不同于针对负载平衡服务设置的检查，并且它们调用的响应失败的健康检查的操作也不同。直接针对[实例组](https://cloud.google.com/compute/docs/instance-groups/#monitoring_groups)设置的健康检查将导致返回健康检查失败的实例删除该实例并启动一个新实例。而针对负载平衡服务设置的健康检查基本上停止将流量定向到健康检查失败的实例，但让实例运行。您不应该通过负载平衡服务设置与直接针对实例组(这里是 dragons)相同的检查。

在上述两种情况下，您都设置了超时标志值。(我知道我已经转移到健康检查上了，但是我没有忘记这篇文章的核心是关于诚实的！)

用于设置[健康检查](https://cloud.google.com/compute/docs/reference/latest/httpHealthChecks)的 gcloud 命令类似于以下示例，然后您可以将该健康检查与负载平衡服务相关联，或者直接针对托管实例组:

```
$ gcloud compute http-health-checks create example-check — port 80 \ — check-interval 10s \ — healthy-threshold 1 \ — timeout 5s \ — unhealthy-threshold 3
```

这将每 10 秒运行一次简单的 http 检查。healthy-threshold 指示在实例被标记为永久不健康之前，必须有多少后续健康检查返回健康状态。healthy-threshold 表示有多少次连续失败会导致实例被标记为不正常。

超时是在返回失败之前等待请求响应的时间(在我们的示例中设置为 5 秒)。超时值不应大于检查间隔(可以有相同的值)。

然后，您可以将您创建的健康检查资源与一个[目标池](https://cloud.google.com/compute/docs/load-balancing/network/target-pools#add_or_remove_a_health_check_from_a_target_pool)或[后端服务](https://cloud.google.com/compute/docs/load-balancing/http/backend-service#create_a_backend_service)相关联。如果您希望它与负载平衡服务相关联。要将支票与[组](https://cloud.google.com/compute/docs/instance-groups/#monitoring_groups)直接关联，您可以使用 set-autohealing 标志

```
$ gcloud beta compute instance-groups managed set-autohealing example-group \ — http-health-check example-check \ — initial-delay 120
```

**initial-delay** 标志允许实例开始和结束运行它们的启动脚本，以便受管实例组不会过早地重新创建该实例，即防止超时的无限循环。

## 应用程序引擎和超时

使用 App Engine standard 访问云存储时，您需要使用 App Engine UrlFetch 功能。云存储客户端库处理应用引擎端和云存储端的超时错误，自动执行重试，因此您的应用程序不需要添加逻辑来处理这一点。超时和重试机制的配置通过 [RetryParams](https://cloud.google.com/appengine/docs/python/googlecloudstorageclient/retryparams_class) 类公开，您可以使用该类在应用程序范围的基础上更改任何或所有默认设置，或者针对 Google 云存储客户端库函数的特定调用(copy2、delete、listbucket、open、stat)查看[文档](https://cloud.google.com/appengine/docs/python/googlecloudstorageclient/retryparams_class)

App Engine 将资源使用限制在一定范围内，以防止资源过度使用，并在同一集群中运行的应用之间提供更好的隔离。App Engine 请求计时器([Java](https://cloud.google.com/appengine/docs/java/#Java_The_request_timer)/[Python](https://cloud.google.com/appengine/docs/python/#Python_The_request_timer)/[Go](https://cloud.google.com/appengine/docs/go/#Go_The_request_timer))确保请求具有有限的生命周期，并且不会陷入无限循环。目前，对前端实例的请求的截止时间是 60 秒。(后端实例没有相应的限制。) .如果请求未能在 60 秒内返回，将引发 DeadlineExceededError。这需要被捕获，有很多方法可以减轻抛出 DeadlineExceededError 或超时的影响。这篇[文章](https://cloud.google.com/appengine/articles/deadlineexceedederrors)雄辩地解释了各种选项，所以如果你正在使用 App Engine，我建议你暂停一下(看我在那里做了什么！)来读一读。