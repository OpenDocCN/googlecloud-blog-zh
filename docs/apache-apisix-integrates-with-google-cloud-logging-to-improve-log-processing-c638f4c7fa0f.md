# Apache APISIX 与 Google Cloud Logging 集成，以改进日志处理

> 原文：<https://medium.com/google-cloud/apache-apisix-integrates-with-google-cloud-logging-to-improve-log-processing-c638f4c7fa0f?source=collection_archive---------0----------------------->

![](img/da8151b60488fb69cc40fae9160db2ce.png)

日志记录是分布式系统的重要基础设施。可以帮助开发者观察服务运行的状态，提高服务故障排查和诊断的效率，进行多维度分析，提高系统的整体稳定性和运行效率。

[Google Cloud Logging](https://cloud.google.com/logging/) 是 Google Cloud 提供的实时日志管理服务，提供 EB 级的存储、搜索、分析、告警服务。Google Cloud Logging 的日志浏览器可以让您轻松高效地搜索、排序和分析日志，Google Cloud Logging 还提供了保存的查询和丰富的图形功能，使日志筛选结果可检索且更加直观。

Apache APISIX 此前已经支持集成 [HTTP Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/http-logger.md) 、 [TCP Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/tcp-logger.md) 、 [Kafka Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/kafka-logger.md) 、 [UDP Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/udp-logger.md) 、 [RocketMQ Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/rocketmq-logger.md) 、 [SkyWalking Logger](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/skywalking-logger.md) 、[阿里云云日志(SLS)](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/sls-logger.md) 等众多开源和云日志服务解决方案。

最近，Apache APISIX 还增加了对 Google Cloud Logging 的支持，为用户在使用 Apache APISIX 作为网关时提供了新的日志解决方案:使用 [google-cloud-logging](https://github.com/apache/apisix/blob/master/docs/en/latest/plugins/google-cloud-logging.md) 将 Apache APISIX 请求日志转发到 Google Cloud Logging 服务进行分析和存储。

当插件启用时，Apache APISIX 将在日志阶段获取请求上下文信息，并将其序列化为 Google Cloud Logging 的 [LogEntry](https://cloud.google.com/logging/docs/reference/v2/rest/v2/LogEntry) ，然后将序列化的日志数据提交给批处理队列，当批处理队列触发用户设置的时间或条目阈值时，日志数据将通过 Google Cloud API 转发给 Google Cloud Logging service。

本文将解释如何在 Apache APISIX 中配置和使用 Google Cloud Logging 服务。

# 配置谷歌云

*   打开浏览器，访问[谷歌云主页](https://console.cloud.google.com/home/dashboard)。
*   输入您的用户名和密码以登录 Google Cloud 控制台。
*   点击 Google Cloud 控制台左侧菜单，选择“IAM & Admin >创建项目”开始创建项目。

![](img/fcdfff5326c8837179194ab05755bbc7.png)

*   输入项目名称，选择组织名称，然后单击“创建”创建项目。

![](img/431b808ad3d2f24dd6758fb60d022994.png)

*   成功创建项目后，控制台的右上角会显示创建成功。

![](img/ac1407ee5dfa38d91aecc3e04e6441e1.png)

*   在窗口中单击选择项目，或者在控制台主页的顶部导航栏中选择项目操作路径。选择项目后，您将被重定向到控制台主页，在这里您已经可以在顶部导航栏中看到有关当前项目的数据，并在信息中心中看到项目信息。

![](img/3c1d89e4c2abc910f7b1b9a2bee7bfee.png)

*   创建完项目后，您需要为项目创建一个服务帐户。请返回 Google Cloud console 主页，点击左侧菜单中的“IAM & Admin >服务帐户”,开始创建服务帐户。

![](img/f20adbad8a70bf366a569dab44bae2c7.png)

*   单击“创建服务帐户”创建服务帐户。

![](img/41e7a2efced0b356b8645f23966f4f83.png)

*   输入服务帐户名和 ID(ID 通常跟在帐户生成之后)，然后单击“创建并继续”。

![](img/059aa0d7f656ea15cd694a714a55693c.png)

*   单击“Role ”,在搜索框中键入“Logging Admin ”,搜索该角色，并选择“Logging Admin”作为角色。

![](img/1089525424a516118baf7f20e56f53ac.png)

单击“完成”以完成服务帐户创建，并跳转到服务帐户主页。此时，您可以在列表中看到您刚刚创建的帐户及其详细信息。

![](img/a4ab64f409db87be4d4cb307e23f882c.png)

*   点击服务账号最后一栏的“管理密钥”进入密钥管理界面。

![](img/dc37d9e8109e4df08a7900bef593e1f9.png)

*   点按“添加密钥”>“创建新密钥”以开始创建新的密钥。

![](img/4eb23f920a820c49de5ed8c79150d217.png)

*   在弹出的页面中选择密钥类型为“JSON”，然后点击“创建”创建一个新的密钥。

![](img/f9f0e26c02dbca129b73f6c33ce67e1f.png)

*   私钥信息将通过您的浏览器自动下载到系统默认下载目录。当您启用 google-cloud-logging 插件时，您需要使用此私钥中的信息，因此请保存私钥文件。

![](img/f48a9240fdeef3bf49f576f223598d44.png)

# 配置 Apache APISIX

## 启用谷歌云日志插件

**选项 1:上传密钥文件配置**

1.  将私钥文件上传到 Apache APISIX 节点服务器。
2.  配置`google-cloud-logging. auth_file`配置项的文件路径。

```
curl [http://127.0.0.1:9080/apisix/admin/routes/1](http://127.0.0.1:9080/apisix/admin/routes/1)  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '{ "uri":"/logging.do", "upstream":{ "type":"roundrobin", "nodes":{ "127.0.0.1:1980":1 } }, "plugins":{ "google-cloud-logging":{ // Google Cloud Logging Private Key File "auth_file":"/path/to/apache-apisix-fcafc68c2f41.json", // Maximum number of entries per batch queue. "batch_max_size": 1, // Maximum time to refresh the buffer in seconds. "inactive_timeout": 10 } }}'
```

**选项 2:在 JSON 中声明配置**

1.  打开私钥文件。
2.  将“项目标识”的值配置为`google-cloud-logging. auth_config.project_id`。
3.  将“私有密钥”的值配置为`google-cloud-logging. auth_config. private_key`。

```
curl [http://127.0.0.1:9080/apisix/admin/routes/1](http://127.0.0.1:9080/apisix/admin/routes/1)  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '{ "uri":"/logging.do", "upstream":{ "type":"roundrobin", "nodes":{ "127.0.0.1:1980":1 } }, "plugins":{ "google-cloud-logging":{ // Google Cloud Logging Private Key File "auth_config":{ "project_id":"apache-apisix", "private_key":"-----BEGIN RSA PRIVATE KEY-----your private key-----END RSA PRIVATE KEY-----" }, // Maximum number of entries per batch queue. "batch_max_size": 1, // Maximum time to refresh the buffer in seconds. "inactive_timeout": 10 } }}'
```

**参数**

插件参数可以在[这里](https://apisix.apache.org/blog/2021/12/22/google-logging/#parameters)找到。

## 验证插件运行正常

1.  运行以下命令向 Google Cloud Logging 发送请求。
2.  打开浏览器，访问[谷歌云主页](https://console.cloud.google.com/home/dashboard)。
3.  输入您的用户名和密码以登录 Google Cloud 控制台。
4.  查看通过日志浏览器发送的请求日志，返回的结果如下所示。

```
curl -i http://127.0.0.1:9080/logging.doHTTP/1.1 200 OKContent-Type: text/html; charset=utf-8Transfer-Encoding: chunkedConnection: keep-aliveDate: Fri, 10 Dec 2021 09:57:52 GMTServer: APISIX/2.11.0Hello, Google Cloud Logging
```

![](img/e8cf821755371ec204faf787e64f895f.png)

## 禁用谷歌云日志插件

如果您使用完插件，您可以移除`google-cloud-logging`相关的配置块来停用插件。

```
curl [http://127.0.0.1:9080/apisix/admin/routes/1](http://127.0.0.1:9080/apisix/admin/routes/1)  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '{ "uri":"/logging.do", "upstream":{ "type":"roundrobin", "nodes":{ "127.0.0.1:1980":1 } }, "plugins":{ }}'
```

# 摘要

本文描述了连接 Apache APISIX 和 Google Cloud Logging 的详细步骤。我们希望这篇文章能让你对在 Apache APISIX 中使用 Google Cloud Logging 有一个更清晰的认识，方便后续的动手操作。

Apache APISIX 不仅致力于保持自身的高性能，也一直非常重视开源生态的建设。目前，Apache APISIX 拥有 10+个日志相关插件，支持与业界主流开源日志项目接口。

如果您需要与其他日志进行交互，请访问 Apache APISIX 的 [GitHub](https://github.com/apache/apisix/issues) 并通过 issue 留下您的建议；或者订阅 Apache APISIX [邮件列表](https://apisix.apache.org/zh/docs/general/subscribe-guide)，通过邮件表达自己的想法。