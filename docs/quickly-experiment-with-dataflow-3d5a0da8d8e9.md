# 如何快速试验数据流(Apache Beam Python)

> 原文：<https://medium.com/google-cloud/quickly-experiment-with-dataflow-3d5a0da8d8e9?source=collection_archive---------1----------------------->

我的一个同事向我展示了这个技巧，让我快速试验[云数据流](https://cloud.google.com/dataflow/) /Apache Beam，它已经为我节省了几个小时。[数据流是谷歌的自动缩放，无服务器处理批处理和流数据的方式。它运行[阿帕奇光束](https://beam.apache.org/)管道。如果你没有用过，你应该尝试一下。]

为了尝试一些 Python 数据流代码，我会这样做:我会创建一个管道，从一个 CSV 文件中读取一些数据，用我正在尝试的代码转换它，将结果写出到一个文本文件中，然后查看它。非常非常慢的过程。

这个很酷的新方法利用了 Python [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) (命令行解释器)以及 Python 列表可以作为数据流来源的事实。

如有必要，在您的机器上安装 Apache Beam 包:

```
$pip install 'apache-beam[gcp]'
```

在命令行上启动 Python 解释器:

```
$ python
```

导入 Apache Beam 包:

```
>>> import apache_beam as beam
```

现在，你可以开始了。您可以创建一个示例列表，并将其传递给转换:

```
>>> [3, 8, 12] | beam.Map(lambda x : 3*x)[9, 24, 36]
```

多酷啊。没有管道，没有输入/输出文件。只是一个简单的列表，通过管道连接到您想要尝试的转换代码。

下面是一个在键-值对(在 Python 数据流中表示为 2 元组)上进行尝试的例子:

```
>>> [(‘Jan’,3), (‘Jan’,8), (‘Feb’,12)] | beam.GroupByKey()[(‘Jan’, [3, 8]), (‘Feb’, [12])]
```

您可以继续追加变换:

```
>>> [(‘Jan’,3), (‘Jan’,8), (‘Feb’,12)] | beam.GroupByKey() | beam.Map(lambda (mon,days) : (mon,len(days)))[(‘Jan’, 2), (‘Feb’, 1)]
```

希望这个技巧能为你节省和我一样多的时间。

编码快乐！