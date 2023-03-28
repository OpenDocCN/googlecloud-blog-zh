# 如何使用云功能自动连接 Google 云存储上的分片文件

> 原文：<https://medium.com/google-cloud/how-to-write-to-a-single-shard-on-google-cloud-storage-efficiently-using-cloud-dataflow-and-cloud-3aeef1732325?source=collection_archive---------0----------------------->

在 Apache Beam 中，当您使用以下命令将文本文件写出到 blob 存储时:

```
beam.io.WriteToText('gs://somebucket/somedir/csv/train')
```

您将得到许多输出碎片，如下所示:

```
gs://somebucket/somedir/csv/train-00214-of-00262
gs://somebucket/somedir/csv/train-00215-of-00262
gs://somebucket/somedir/csv/train-00216-of-00262
```

确切的数字取决于您的数据集、工人数量等。然而，很多时候，您会想要一个输出文件，因为您使用的软件只想要一个文件。

简单的方法是告诉 Beam 只写出一个 shard:

```
beam.io.WriteToText(options.output_prefix, **num_shards=1**)
```

然而，这是非常低效的。分布式系统的强大之处在于能够完全并行化工作，并且拥有单个接收器会降低您的束管道的速度。

![](img/100735c213d17bb968ddbba6a7551b13.png)

我希望理论是正确的，但我现在在 GitHub 中有我的例子，所以谁知道呢？(xkcd 的漫画)

我最近面临这个问题，用云函数解决了(GitHub 中的[代码)。](https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/blogs/inference/flights/compose_shards.py)

## 写入碎片

在梁管线中，指定一些碎片。我们将使用的 Google 云存储的“合成”功能目前有 32 个文件的限制，所以你不希望你的管道产生更多的文件。同时，为了提高效率，您希望拥有比数据流作业中的工人数量更多的碎片:

```
beam.io.WriteToText(options.output_prefix, num_shards=10)
```

完成后，您将拥有一堆文件。你如何连接它们？

## 云存储构成

在 GCS 上连接一堆文件的简单方法是将它们下载到一个 VM 上，用 Unix 的“cat”连接它们，然后上传。不要那样做！

谷歌云存储支持一个叫做[“合成”](https://cloud.google.com/storage/docs/json_api/v1/objects/compose)的漂亮功能:它让你用多达 32 个源 blob 合成一个 blob。您可以通过命令行执行以下操作来实现这一点:

```
gsutil compose \
  gs://${BUCKET}/somedir/csv/train* \
  gs://${BUCKET}/somedir/csv/full_training_data.csv
```

无需下载或本地存储！您也可以在合成后删除碎片，以避免混乱。因此，创建单个输出文件的有效方法是运行数据流作业，告诉它产生多达 32 个碎片，然后使用合成功能连接输出。

嗯，您可以运行数据流作业，等待它完成，然后调用 compose 命令。但是，当您可以设置一个云函数，每当分片文件出现在桶中时，它就会自动为您完成这项工作，为什么还要这样做呢？

## 云函数做作曲

我转到 GCP web 控制台，导航到云函数控制台，在我的 bucket 中创建一个函数来触发创建事件，选择 Python 3.7 作为我的首选语言，然后在 main.py 中键入以下 Python 函数:

```
import google.cloud.storage.client as gcs
import loggingdef compose_shards(data, context):
  num_shards = 10
  prefix = 'somedir/csv/train'
  outfile = 'somedir/csv/full_training_data.csv'
  # trigger on the last file only
  filename = data['name'] last_shard = '-%05d-of-%05d' % (num_shards - 1, num_shards)
  if (prefix in filename and last_shard in filename):
    # verify that all 10 shards exist
    prefix = filename.replace(last_shard, '')
    client = gcs.Client()
    bucket = client.bucket(data['bucket'])
    blobs = []
    for shard in range(num_shards):
      sfile = '%s-%05d-of-%05d' % (prefix, shard + 1, num_shards)
      blob = bucket.blob(sfile)
      if not blob.exists():
        # this causes a retry in 60s
        raise ValueError('Shard {} not present'.format(sfile))
      blobs.append(blob)
    # all shards exist, so compose
    bucket.blob(outfile).compose(blobs)
    logging.info('Successfully created {}'.format(outfile))
    for blob in blobs:
      blob.delete()
    logging.info('Deleted {} shards'.format(len(blobs)))
```

我还勾选了失败时“重试”的复选框，并在 requirements.txt 中指定了“google-cloud-storage”包。

上面的代码是做什么的？它验证它是在具有我的数据流管道正在写入的前缀的文件上触发的，并且这是最后一个碎片。

现在，最后一个碎片不一定是最后一个被写出的。可能存在争用情况，或者某个早期文件可能非常大。所以，我做了一点防御性编程，确保我期望的所有 10 个碎片都存在。我可以通过用期望的文件名构造一个 Blob 并检查它是否存在来实现。如果不存在，我就抛出一个异常。回想一下，我们设置了这个函数，如果是这种情况，它将在 60 秒后重试。

一旦我确认所有的碎片都存在，我就可以使用 API 调用将 blob 组合成一个大 blob，并删除单个 blob。

## 下午茶时间

所以，现在，当我运行我的数据流管道时，它产生了 10 个斑点。一旦第 10 个 blob 到达 Google 云存储，Cloud 函数就会运行，并将所有 10 个 blob 连接成一个 blob。

我吗？我可以在作业运行时去喝茶，而函数会监视作业。