# 如何在云日志中存储日志

> 原文：<https://medium.com/google-cloud/how-logs-are-stored-in-cloud-logging-b6869ced0fa?source=collection_archive---------1----------------------->

这篇文章是[云日志如何工作](https://minherz.medium.com/how-cloud-logging-works-series-3aab7e7a1eed)系列的一部分。

![](img/15bb222a82845b171d1978fdb0d6a792.png)

云日志以 *protobuf* [二进制格式](https://developers.google.com/protocol-buffers/docs/encoding)存储所有日志。如果日志导出到其他存储位置(例如 BigQuery ),格式可能会有所不同。用户添加到条目中的定制标签或 Json 有效负载元素越多，存储效率就越低。这是因为对于定制元素(比如键:值对)，protobuf 必须存储键和值，而对于“已知”字段，它只存储值。

如果您需要估计您将需要的日志存储容量，并且将为此计费，您可以使用这些知识(有关详细信息，请参见[云操作套件定价](https://cloud.google.com/stackdriver/pricing))。

每个日志条目的大小被[限制](https://cloud.google.com/logging/quotas#log-limits)为大约。256KB。如果您发送日志条目大于 256KB 的摄取[请求](https://cloud.google.com/logging/docs/reference/v2/rest/v2/entries/write)，它将被拒绝。如果您发送多个日志条目，尽管可能有一些无效条目，但您希望所有“好的”条目都被接收，您应该使用请求的`partialSuccess` [参数。当它设置为`true`时，云日志后端将存储所有有效的日志条目，并将返回未存储的无效条目列表，并说明原因。最后但同样重要的是，注意请求的总大小。也是有限的。日志接收请求的大小不能大于 10MB。](https://cloud.google.com/logging/docs/reference/v2/rest/v2/entries/write#body.request_body.FIELDS.partial_success)