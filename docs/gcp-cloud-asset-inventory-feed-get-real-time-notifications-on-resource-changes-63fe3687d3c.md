# GCP 云资产清单馈送:获得关于资源变化的实时通知

> 原文：<https://medium.com/google-cloud/gcp-cloud-asset-inventory-feed-get-real-time-notifications-on-resource-changes-63fe3687d3c?source=collection_archive---------0----------------------->

在拥有许多云资产的组织中，跟踪整个资源清单是一项挑战。可以分别为组织中的每个项目列出所有资源的列表，但是这很费时间，容易出错，而且很乏味。即使您能够获得组织中所有项目的所有资源列表，维护这些资源如何变化也将是一场噩梦。在任何给定的时间获取资源的状态几乎是不可能的。保持对每个资源是否遵守企业的安全性和法规遵从性标准的检查变得更加具有挑战性。

这就是 GCP [云资产清单](https://cloud.google.com/asset-inventory/docs/overview)可以提供帮助的地方。云资产清单会随时跟踪您的 GCP 资源。该数据库保存了库存中每项资产 5 周的元数据历史记录，并允许您随时查询库存。

以下是云资产清单的一些显著特征:

1.  搜索资产元数据
2.  在任何给定的时间戳导出清单，或导出特定时间范围内的更改历史。
3.  使用资产源订阅资产变更的实时通知。
4.  IAM 策略分析器检查谁可以访问什么。

在本文中，我们将了解如何订阅资产变化的实时通知，并在创建了具有公共 IP 的 Google 计算引擎实例时收到警报。

# 目录

1.  体系结构
2.  先决条件
3.  创建资产馈送
4.  设置警报
5.  测试和验证
6.  打扫
7.  结论

# 1.体系结构

下图显示了该解决方案的高级架构。

![](img/802521eebead116783725fb30cbaabb1.png)

我们将使用云资产清单、发布/订阅和云功能来生成实时通知。

当使用公共 IP 创建计算引擎实例时，资产清单会实时捕获这种变化。通过资产馈送，该信息被发布到发布/订阅主题。一个云功能，订阅发布/订阅主题，并向一个已配置的松弛通道发送通知。

# 2.先决条件

首先，在您的项目中启用云资产、发布/订阅、资源管理器和云函数 API。

```
gcloud services enable cloudasset.googleapis.com pubsub.googleapis.com cloudfunctions.googleapis.com cloudresourcemanager.googleapis.com
```

现在，让我们创建一个 PubSub 主题，它将成为资产提要的目标。

```
gcloud pubsub topics create gce-public-ip-feed-topic --project=**PROJECT_ID**
```

# 3.创建资产馈送

现在让我们创建一个资产提要。摘要是您可以订阅的更改流。每个级别(项目、文件夹、组织)最多可以有 200 个提要。

因此，让我们创建一个 feed，它将获得计算实例创建的实时更改。

```
gcloud asset feeds create gce-public-ip-feed --project=**PROJECT_ID** \
--content-type=resource \
--asset-types="compute.googleapis.com/Instance" \
--pubsub-topic="projects/**PROJECT_ID**/topics/gce-public-ip-feed-topic" \ 
--condition-expression="temporal_asset.prior_asset_state == google.cloud.asset.v1.TemporalAsset.PriorAssetState.DOES_NOT_EXIST"
```

> *注意:资产馈送只能通过 CLI 和 API 创建。通过云控制台创建它的选项不可用。*

有关参数参考，请查阅[g 云资产源](https://cloud.google.com/sdk/gcloud/reference/asset/feeds/create)的文档。

有关支持的资产类型列表，请参考此处的文档。

默认情况下，对资源的任何更改(如创建/更新/删除)都会导致消息发布到发布/订阅主题。我们可以使用`--condition-expression` 标志来过滤对底层资产类型的任何特定更改。在上面的例子中，我只过滤了新实例的创建。这里可以参考使用`--condition-expression` [的文档。](https://cloud.google.com/asset-inventory/docs/monitoring-asset-changes-with-condition)

您还可以使用`--asset-name`标志为提要指定特定资源的名称。

`--pubsub-topic`是我们之前创建的主题。它将成为此源的目标。

使用以下命令检索创建的提要:

```
gcloud asset feeds describe gce-public-ip-feed --project=PROJECT_ID
```

现在，我们已经准备好使用变更并发送通知了。

> *注意:创建后可能需要 10 分钟，提要才会开始向发布/订阅主题发送消息。*

# 4.设置警报

要向松弛通道发送通知，请执行以下操作:

1.  创建一个新的 Slack 应用程序，该应用程序有足够的权限将消息发布到公共 Slack 频道。使用此处的说明[创建一个 app，并生成一个`TOKEN`用于云端功能，将消息发布到频道。](https://api.slack.com/authentication/basics)
2.  创建并部署一个云功能，当收到来自 Pub/Sub 的通知时，该功能会将聊天消息发布到 Slack。

在这一节中，我们将设置一个云函数，它从我们的 PubSub 主题中读取消息，并在使用 Public IP 创建实例时发出一条 slack 消息。

首先，让我们为云函数代码创建一个目录:

```
mkdir -p alert_function && cd alert_function
```

现在创建两个文件`requirements.txt`和`main.py`

**requirements.txt**

```
requests==2.26.0
```

**main.py**

```
import base64
import json
import requests

# Triggered from a message on a Cloud Pub/Sub topic.
def filter_rule(event, context):
    """Triggered from a message on a Cloud Pub/Sub topic.
    Args:
         event (dict): Event payload.
         context (google.cloud.functions.Context): Metadata for the event.
    """
    TOKEN = "xoxb-XXXXXXXXXXX-YYYYYYYY-ZZZZZZZ"
    URL = "https://slack.com/api/chat.postMessage"

    pubsub_message = base64.b64decode(event['data']).decode('utf-8')
    message_json = json.loads(pubsub_message)
    natIP = json_extract(message_json,"type")

    if 'ONE_TO_ONE_NAT' in natIP:
        payload = get_data(message_json)
        name = payload["name"]
        requests.post(URL, data={
            "token": TOKEN,
            "channel": "#general",
            "text": f"Instance *{name}* was created with external IP!"
        })

def get_data(message_json):
    try:
        return message_json["asset"]["resource"]["data"]
    except:
        return message_json["priorAsset"]["resource"]["data"]

def json_extract(obj, key):
    """Recursively fetch values from nested JSON."""
    arr = []
    def extract(obj, arr, key):
        """Recursively search for values of key in JSON tree."""
        if isinstance(obj, dict):
            for k, v in obj.items():
                if isinstance(v, (dict, list)):
                    extract(v, arr, key)
                elif k == key:
                    arr.append(v)
        elif isinstance(obj, list):
            for item in obj:
                extract(item, arr, key)
        return arr
    values = extract(obj, arr, key)
    return values 
```

将变量`TOKEN`替换为您自己的 Slack App Bot 身份验证令牌，如上面步骤 1 中所创建的。

现在部署这个云功能。

```
gcloud functions deploy gce-public-ip-alert --region=us-central1 --entry-point=filter_rule --runtime=python39 --trigger-topic=gce-public-ip-feed-topic
```

> *注意:如果尚未启用云构建服务，系统会提示您启用该服务。*

一旦你的功能处于`Active`状态。我们准备测试和验证我们的设置。

# 5.测试和验证

让我们在项目中创建一个带有 public-ip 的计算引擎实例，用于测试目的

```
gcloud compute instances create test-instance --project=PROJECT_ID --network=NETORK_ID --subnet=SUBNET_ID --zone=us-central1-a
```

如果一切都设置正确，你会在你的松弛频道看到一条消息。

![](img/ce8e7a7d1eca376939f822ac7a2cb91d.png)

关于时差的通知

# 6.打扫

如果您在本演示中使用了一个新项目，那么您可以删除该项目来清理资源。否则，按照以下步骤删除创建的资源-

删除实例

```
gcloud compute instances delete test-instance --project=PROJECT_ID
```

删除云功能

```
gcloud functions delete gce-public-ip-alert --project=PROJECT_ID
```

删除资产馈送

```
gcloud asset feeds delete gce-public-ip-feed --project=PROJECT_ID
```

删除发布/订阅主题

```
gcloud pubsub topics delete gce-public-ip-feed-topic
```

# 7.结论

Google Cloud Asset inventory 是一个强大的工具，可以帮助组织跟踪云中的所有资源。通过资产馈送，他们可以获得关于其环境中发生的变化的实时警报，并使用云功能采取适当的措施。

**脚注:**

有几个选项可以实现类似的功能:

1.  在云日志中创建基于日志的指标，并在其上创建警报。
2.  默认情况下，使用组织策略约束在 GCE 虚拟机实例上禁用公共 IP，仅在特定虚拟机上允许。更多细节可以在[这里](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#disableexternalip)找到。