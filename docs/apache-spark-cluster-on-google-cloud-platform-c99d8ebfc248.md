# Google 计算引擎实例上的 Apache Spark 集群

> 原文：<https://medium.com/google-cloud/apache-spark-cluster-on-google-cloud-platform-c99d8ebfc248?source=collection_archive---------0----------------------->

![](img/0a42eaf2b57ce4c829a2a5d022d79168.png)

最近我想到用我的免费谷歌虚拟机[**的 8 核和 52 GB 内存来运行 Apache Spark 集群。互联网上有几个在物理机上设置 Spark 的指南，但没有一个完全适用于计算引擎。本指南试图提供在计算引擎实例上设置 Spark 的直接、逐步解决方案。**](https://cloud.google.com/free-trial/)

> **注意:**我有一个…