# 使用数据流 SQL 在 Google Cloud 上丰富基本流数据

> 原文：<https://medium.com/google-cloud/basic-streaming-data-enrichment-on-google-cloud-with-dataflow-sql-a7684353119c?source=collection_archive---------0----------------------->

存在许多技术来丰富数据，尽管有一种技术可以与 SQL 这样的简单语言一起工作，同时允许您进行批处理和流处理，但这种技术很少，其中之一是 Google Cloud 上的数据流。

# 什么是阿帕奇光束？

> Apache Beam 是一个统一的模型，用于定义批处理和流数据并行处理管道，以及一组特定于语言的 SDK，用于构建管道和运行程序，以便在分布式环境中执行它们