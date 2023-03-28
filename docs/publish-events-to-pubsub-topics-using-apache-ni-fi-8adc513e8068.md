# 使用 Apache Ni-Fi 将事件发布到发布订阅主题

> 原文：<https://medium.com/google-cloud/publish-events-to-pubsub-topics-using-apache-ni-fi-8adc513e8068?source=collection_archive---------1----------------------->

Apache Nifi 是一个实时事件处理平台，可用于不同系统之间的数据移动。

NiFi 还可以用于连接云服务，如 AWS、Google cloud 或 Microsoft Azure，使用内置处理器在云平台之间移动数据。

下面的故事是关于每当在 google 云存储桶中创建新对象时，将实时事件发布到 google cloud pub-sub。

PubSub 是 google cloud 上的一个发布和订阅平台，可以用于事件处理，也可以像其他消息传递系统一样用作系统之间的数据通信通道。

以下步骤可用于构建工作流，以便将 google 云存储的 OBJECT_FINALIZE 事件发布到发布订阅主题。

使用案例:

从配置单元表中选择数据，转换文件属性，如文件名、文件格式(Avro/Csv ),将文件写入 GCS 存储桶，并向发布子主题发送通知，其中包含消息属性，如(文件名、表名、行数、文件创建日期等)

1.  使用你的 console.cloud.google.com 在谷歌云发布订阅中创建一个主题
2.  通过将以下处理器拖到 nifi 画布上来创建一个 NiFi 流

选择 HiveQl，更新属性，放置对象

3.添加所需的控制器配置:

a)hivecontroller 服务，用于连接到您的 hive 服务器和数据库。

b)GCSController 服务，用于连接到您的 google 云存储实例

4)从下面下载自定义 pubsub 处理器代码，构建 nar 包并将其添加到 nifi lib 文件夹中

5)将上述处理器添加到画布中，并将其连接到 putgcsobject 处理器。在“配置”选项卡中添加主题名称和项目配置

![](img/48848107af38320edc1bb2155878f50d.png)

Nifi 发布流

上面的流程将把 Json 事件发布到 GCPPubSubPublisher 处理器中提到的 pubsub 主题，文件的元数据写入存储桶。(它不会将文件内容发送到 pubsub，而是只发送事件通知，告知桶中有新的文件或对象可供使用或处理)