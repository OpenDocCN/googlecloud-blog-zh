# 从云功能启动/停止计算引擎实例

> 原文：<https://medium.com/google-cloud/start-stop-compute-engine-instance-from-cloud-function-bf9ae5199609?source=collection_archive---------0----------------------->

我设计了一个运行在 Google 计算引擎上的 web 服务来完成一些长时间的数据处理任务。数据将存储在云存储中，计算引擎上的服务将从云存储中获取数据。处理后，服务会将处理后的数据保存回云存储。并且，该服务没有被频繁使用。

如果我们启动一个计算引擎实例，它将开始消耗我们的资源。为了节省成本，我需要尽量减少浪费。因此，我开始调查是否有办法只在我们需要时才启动计算引擎。

从谷歌云平台文档中挖掘后，我发现:

 [## 使用 Node.js 客户端库|计算引擎文档| Google 云平台

### 在计算引擎上部署 Docker 容器

cloud.google.com](https://cloud.google.com/compute/docs/tutorials/nodejs-guide) 

这个库允许我们访问计算引擎！我决定改变我的系统设计:当数据上传到云存储时，它触发云功能。云函数发送命令启动计算引擎处理数据。处理后，计算引擎触发云函数事件来停止计算引擎本身。通过这种设计，我们可以在不需要使用虚拟机时停止虚拟机，以节省资金。也就是说，只在需要时才启用虚拟机！

对于云函数，我创建了两个函数:

启动计算引擎:

```
var http = require('http');var Compute = require('@google-cloud/compute');var compute = Compute();exports.startInstance = function startInstance(req, res) { var zone = compute.zone('***instance zone here***'); var vm = zone.vm('***instance name here***'); vm.start(function(err, operation, apiResponse) { console.log('instance start successfully');
    });res.status(200).send('Success start instance');};
```

package.json:

```
{"name": "sample-http","dependencies": {"@google-cloud/compute": "0.7.1"},"version": "0.0.1"}
```

停止计算引擎:

```
var Compute = require('@google-cloud/compute');var compute = Compute();exports.stopInstance = function stopInstance(req, res) { var zone = compute.zone('***instance zone here***'); var vm = zone.vm('***instance name here***'); vm.stop(function(err, operation, apiResponse) { console.log('instance stop successfully'); }); res.status(200).send('Success stop instance');};
```

测试这些函数，您将看到您的实例开始/停止。这篇文章是我的一些有趣的尝试。对一个真正的系统设计可能没什么用，但对我来说会省钱！

目前，我没有完成设计的完整流程，所以也许在了解更多关于谷歌云平台的信息后，我的服务的设计会再次改变。:)