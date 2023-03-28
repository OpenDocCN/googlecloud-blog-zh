# AWS 到 Google 云平台服务映射

> 原文：<https://medium.com/google-cloud/aws-to-google-cloud-platform-service-mapping-5a1689d41f01?source=collection_archive---------3----------------------->

**2016 年 9 月 1 日**更新:我们现在有一个非常全面的地图，远远超出了这篇文章的范围。你可以在这里阅读[。](https://cloud.google.com/docs/compare/aws/)

作为[谷歌云平台](http://cloud.google.com)团队的开发者倡导者，我经常被问到我们提供什么服务。如果与我交谈的人熟悉亚马逊网络服务(AWS)，那么开始解释谷歌云平台的最快方法是从与 AWS 的类似服务进行比较开始，然后涵盖差异。

下面是 AWS 和 Google 云平台中一些主要服务之间的简单映射。这并不是一个完整的映射。列出每一项服务对两个平台都不公平，因为谷歌和亚马逊在许多领域采取不同的方法，直接比较几乎是不可能的。我只列出了比较有帮助的服务。

## 计算:

[EC2](http://aws.amazon.com/ec2/) → [计算引擎](https://cloud.google.com/compute/)
[EC2 容器服务](http://aws.amazon.com/ecs/) → [容器引擎](https://cloud.google.com/container-engine/)
[弹性豆茎](http://aws.amazon.com/elasticbeanstalk/) → [App 引擎](https://cloud.google.com/appengine/) ******

## 存储:

[S3](http://aws.amazon.com/s3/) → [云存储](https://cloud.google.com/storage/)
[冰川](http://aws.amazon.com/glacier/) → [云存储近线](https://cloud.google.com/storage-nearline/)
[CloudFront](http://aws.amazon.com/cloudfront/)→[云存储](https://cloud.google.com/storage/docs/concepts-techniques)(公共桶提供边缘缓存)

## 数据库:

[RDS](http://aws.amazon.com/rds/) → [云 SQL](https://cloud.google.com/sql/)
DynamoDB→[云 Datastore](https://cloud.google.com/datastore/) 和[云 Bigtable](https://cloud.google.com/bigtable/)

## 大数据:

[红移](http://aws.amazon.com/redshift/)→[big query](https://cloud.google.com/bigquery/)
[SQS](http://aws.amazon.com/sqs/)/[Kinesis](http://aws.amazon.com/kinesis/)→[云 Pub/Sub](https://cloud.google.com/pubsub/)
[EMR](http://aws.amazon.com/elasticmapreduce/)→[云数据流](https://cloud.google.com/dataflow/)

## 监控:

[CloudWatch](http://aws.amazon.com/cloudwatch/) → [云监控](https://cloud.google.com/monitoring/)和[云日志](https://cloud.google.com/logging/docs/)

## 网络:

[Route53](http://aws.amazon.com/route53/) → [云 DNS](https://cloud.google.com/dns/) 和[谷歌域名](http://domains.google.com/)
[直连](http://aws.amazon.com/directconnect/) → [云互联](https://cloud.google.com/networking/#interconnect)

## 其他:

[CloudFormation](http://aws.amazon.com/cloudformation/) → [云部署管理器](https://cloud.google.com/deployment-manager/)
SES→[send grid](https://cloud.google.com/compute/docs/tutorials/sending-mail)(合作伙伴)
[WorkMail](http://aws.amazon.com/workmail/)→[Gmail](http://mail.google.com/)(另见[Google for Work](https://www.google.com/work/))
[Work Docs](http://aws.amazon.com/workdocs/)→[Google Docs](https://www.google.com/docs/about/)(另见 [Google for Work](https://www.google.com/work/) )

****** AWS Elastic Beanstalk 和 Google App Engine 经常被描述为类似的产品，但它们的方法有显著的差异。两者都提供自动伸缩、负载平衡、监控等功能。但是与 App Engine 不同，Elastic Beanstalk 需要原始虚拟机所需的典型系统管理(操作系统更新等)。).Google App Engine 是一个 PaaS，这意味着它是完全托管的，因此所有这些管理任务都由 Google 处理。基本的应用程序引擎设置包括内置服务，如任务队列、Memcache、用户 API 等。

如果您需要未受管理的虚拟机，Google 还有[自动扩展](https://cloud.google.com/compute/docs/autoscaler/)、[负载平衡](https://cloud.google.com/compute/docs/load-balancing/)和[监控未受管理的虚拟机](https://cloud.google.com/monitoring/)作为 [Google 计算引擎](https://cloud.google.com/compute/)的功能。现在还有一种替代的托管模式，作为谷歌应用引擎的一部分，称为[托管虚拟机](https://cloud.google.com/appengine/docs/managed-vms/)。

我的建议是，在进入任何一个平台之前，都要做足功课，彻底理解这些模型。各有各的优势。

在不久的将来，我会有更多的帖子，提供更多关于几个产品的细节。敬请期待！

*原载于 2015 年 5 月 12 日*[*【gregsramblings.com】*](https://gregsramblings.com/temp15/)*。*