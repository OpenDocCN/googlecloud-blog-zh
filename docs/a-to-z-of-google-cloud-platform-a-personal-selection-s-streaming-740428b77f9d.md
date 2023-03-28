# 谷歌云平台的 a 到 Z 个人精选-S 流

> 原文：<https://medium.com/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-s-streaming-740428b77f9d?source=collection_archive---------0----------------------->

我将主要谈论流式传输，我的意思是我不是指处理无界数据集，而是将数据作为从源到接收器的连续数据流进行馈送的过程。

如果您有一个生成数据的过程，并且您不希望在上传数据之前在本地缓冲它，或者如果您希望将结果从计算管道直接发送到 Google 云存储中，那么流式传输非常有用。

GCP 有很多产品可以让你直接向他们传输数据流，也就是说，接受持续的数据流

Pub/Sub —是 GCP 的工作平台，足够灵活，可以从各种来源接收数据，如遥测数据、游戏数据等，然后可以从 Pub/Sub 中删除数据，并将其传递到 BigQuery、云存储、BigTable 等接收器。管道是一个关键的部分。我在这里谈到了 Pub/Sub [，所以我不会在这篇文章中花更多的时间。](/google-cloud/a-to-z-of-google-cloud-platform-a-personal-selection-p-pub-sub-130538dab6e5#.v2mudj5uc)

云存储——基于 HTTP 分块传输编码，使用 gsutil 工具或 boto 库支持[流传输](https://cloud.google.com/storage/docs/streaming)。流式数据允许您在云存储帐户可用时立即将数据传入传出，而无需先将数据保存到单独的文件中。许多场景要求数据最终存储在云存储中，例如，如果你想将计算管道的结果直接发送到谷歌云存储中，那么流式传输就是实现这一点的方法。

文档中的例子在这方面很棒，所以使用那里的例子来说明如何使用 gsutil 或 boto。

使用 gsutil 将数据传输到管道中使用 gsutil cp 命令，并用破折号替换要复制的文件。以下示例显示了一个名为 collect_measurements 的流程，其输出被传输到一个名为 data_measurements 的 Google 云存储对象:

```
collect_measurements | gsutil cp -gs://my_app_bucket/data_measurements
```

使用 boto 传输数据流时，您可以使用以下命令:

```
dst_uri = boto.storage_uri(*<bucket>* + ‘/’ + *<object>*, ‘gs’)
dst_uri.new_key().set_contents_from_stream(*<stream object>*)
```

因此，用法示例如下:

```
filename = ‘data_file’
MY_BUCKET = ‘my_app_bucket’
my_data = open(filename, ‘rb’)
dst_uri = boto.storage_uri(MY_BUCKET + ‘/’ + filename, ‘gs’)
dst_uri.new_key().set_contents_from_stream(my_data)
```

这会将名为 data_file 的文件流式上载到同名的对象:

虽然这个例子演示了打开一个文件来获取输入流，但是您可以使用任何流对象来代替上面的 my_data。

你也可以在云存储源的地方进行流媒体下载

大查询——通过使用 tabledata()，您可以[将数据一次一条记录地流入大查询](https://cloud.google.com/bigquery/streaming-data-into-bigquery)。insertAll()方法。这种方法支持查询数据，而没有运行加载作业的延迟。

使用 Python 下面是一个示例代码片段，摘自说明如何使用 Python 的文档:

```
def stream_row_to_bigquery(bigquery, project_id, dataset_id, table_name, row,num_retries=5):insert_all_data = {‘rows’: [{‘json’: row,# Generate a unique id for each row so retries don’t accidentally# duplicate insert‘insertId’: str(uuid.uuid4()),}]}return bigquery.tabledata().insertAll(projectId=project_id,datasetId=dataset_id,tableId=table_name,body=insert_all_data).execute(num_retries=num_retries)
```

注意以下限制适用于 [**将数据流式传输到 BigQuery**](https://cloud.google.com/bigquery/streaming-data-into-bigquery) 。(直接取自文档)

*   **最大行大小:** 1 MB
*   **HTTP 请求大小限制:** 10 MB
*   **每秒最大行数:**每个表每秒 100，000 行。超过此数量将导致 quota_exceeded 错误。
*   **每个请求的最大行数:** 500
*   **每秒最大字节数:**每个表每秒 100 MB。超过此数量将导致 quota_exceeded 错误

我知道我说过我主要是要谈论流传输，但是那些一直在阅读本系列的人知道，我对没有真正在数据流上花费任何时间感到有点内疚(我在我的发布/订阅帖子的末尾快速打了个招呼)，所以我将在这里添加一篇关于流处理的文章。我们这样说是什么意思？我们正在讨论的*是一个*数据处理引擎，它是根据无限数据集(未绑定)设计的*。*来自 Dataflow/ Apache Beam 团队的 Tyler 在这个伟大的 [101](http://radar.oreilly.com/2015/08/the-world-beyond-batch-streaming-101.html) 中很好地讨论了这个概念。

数据流需要与发布/订阅结合使用，以接受来自设备的数据流。

在你的数据流代码中，你指定了一个发布/订阅，并使用[发布订阅](https://cloud.google.com/dataflow/model/pubsub-io)来读取主题。读取转换

公共汽车。Read transform 不断从 Pub/Sub 流中读取数据，并返回一个表示流中数据的无界字符串集合

设置它只需要几行代码，下面是文档中的一个例子:

```
PipelineOptions options = PipelineOptionsFactory.create();Pipeline p = Pipeline.create(options);// streamData is Unbounded; apply windowing afterward.PCollection<String> streamData =p.apply(PubsubIO.Read.named(“ReadFromPubsub”).topic(“/topics/my-topic”));
```

我对设置如此简单印象深刻。