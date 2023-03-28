# 堆栈驱动程序监控和 Ruby

> 原文：<https://medium.com/google-cloud/stackdriver-monitoring-and-ruby-b073b8b0af5c?source=collection_archive---------2----------------------->

![](img/ba63a75ad5f97790325a1f1e667665e0.png)

本周，我一直在学习如何使用[谷歌云监控](https://github.com/GoogleCloudPlatform/google-cloud-ruby/tree/master/google-cloud-monitoring) gem 访问 Stackdriver 监控。虽然 Stackdriver Monitoring 是一个完全发布的产品，但是用于监控的 Ruby 客户端库仍然处于 alpha 级别的质量。凭良心说，我不建议您将 alpha 质量库用于像监控这样重要的事情。然而，我知道你们中的一些人无论如何都会这样做，所以这里有一些入门的技巧。

## 安装和认证

不出所料，你用`gem install google-cloud-monitoring`安装宝石。监控 gem 依赖于需要 C 扩展的`grpc`。在某些版本的 Linux 和 JRuby 上安装这个 gem 还有一些未解决的问题。

对于下面的例子，我通过我用`gcloud` SDK 设置的凭证进行认证。还有其他几种身份验证方法。关于认证的官方文档是这里的。我的关于认证的博文[更详细地介绍了不同认证方法的方式和原因。](http://www.thagomizer.com/blog/2017/01/11/authenticating-google-cloud-ruby-gems.html)

在使用监控库的任何脚本的开始，您都需要做一些样板设置。

设置项目变量的行只是确保项目按照监控 API 期望的方式格式化。

## 阅读指标

Stackdriver Monitoring 将指标数据称为时间序列。要通过 Ruby gem 读取度量数据，可以使用`MetricServiceClient#list_time_series`方法。这个方法有四个必需的参数:name、filter、interval 和 view。

Name 是我在上面的样板文件中显示的格式化的项目名称。视图指定应该返回哪些数据。有两个可能的值:`FULL`和`HEADERS`。因为我想要实际的数据，所以我使用了`TimeSeriesView::FULL`。filter 参数允许您控制返回什么数据。有许多[支持的过滤器](https://cloud.google.com/monitoring/api/v3/filters)，包括项目、组和资源类型。在这篇文章中，我将查看运行[“Tind-Purr”](http://www.thagomizer.com/blog/2017/03/09/rails-on-app-engine.html)的应用引擎集群的内存使用情况。您可以使用[指标浏览器](https://app.google.stackdriver.com/metrics-explorer)查找可能的指标。使用 metric explorer，我发现我想要看到的指标是“appengine/system/memory/usage”。当在监控 API 中使用这个指标时，我需要通过添加`.googleapis.com`来指定指标的全名，这给了我"appengine.googleapis.com/system/memory/usage"。

最后一个参数是时间间隔。指定时间间隔有点困难。您需要知道自 Unix 纪元以来的开始和结束时间，以秒为单位。在 Ruby 中，最快的方法是在时间对象上调用`to_i`。下面的代码创建了一个当前时间戳的结束时间和一个结束时间前 10 分钟的开始时间。

```
end_time = Time.now.to_i
start_time = end_time - (60 * 10) # 10 minutes
```

一旦我有了开始和结束时间，我就创建一个新的 TimeInterval 对象，然后为开始和结束时间创建一个`Google::Protobuf::Timestamp`对象。

把所有这些放在一起，我得到这个代码来读取一个度量时间序列。

这个调用返回一个`[Google::Monitoring::V3::TimeSeries](https://googlecloudplatform.github.io/google-cloud-ruby/#/docs/google-cloud-monitoring/v0.24.0/google/monitoring/v3/timeseries)`对象。这个对象有一些元数据和一个点数组。每个点都是`[Google::Monitoring::V3::Point](https://googlecloudplatform.github.io/google-cloud-ruby/#/docs/google-cloud-monitoring/v0.24.0/google/monitoring/v3/point)`的一个实例，它有时间信息和值对象。要提取值，您需要知道类型并调用适当的方法。下面的代码为时间序列创建一个包含所有整数值的数组。

```
points = time_series.first.points.map { |p| p.value.int64_value }
```

这是从 Stackdriver Monitoring 中读取指标所需的代码。在以后的博文中，我将向您展示如何从 Ruby 为 Stackdriver Monitoring 编写自定义指标。

*原载于* [*【致命伤】*](http://thagomizer.com/blog/2017/05/11/stackdriver-monitoring-and-ruby.html)