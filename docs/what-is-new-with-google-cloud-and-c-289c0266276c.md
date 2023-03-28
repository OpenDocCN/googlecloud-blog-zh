# 谷歌云和 C++的新特性是什么

> 原文：<https://medium.com/google-cloud/what-is-new-with-google-cloud-and-c-289c0266276c?source=collection_archive---------0----------------------->

卡洛斯·奥赖安和格雷格·米勒

自从我们在 GCP 发布任何关于 C++的进展已经有一段时间了。那是因为我们一直在努力扩充 C++库的集合。我们很高兴地报告，C++拥有 80 多个面向 GCP 的客户端库，包括强大的基于人工智能的服务，如[文本到语音](https://github.com/googleapis/google-cloud-cpp/blob/main/google/cloud/texttospeech#readme)、[云视觉](https://github.com/googleapis/google-cloud-cpp/blob/main/google/cloud/vision#readme)、[语音到文本](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/speech#readme)、[视频智能](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/videointelligence#readme)，以及通用服务，如[秘密管理器](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/secretmanager#readme)、[云调度器](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/scheduler#readme)和[云任务](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/tasks#readme)。这补充了我们现有的用于 [Bigtable](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/bigtable#readme) 、 [Pub/Sub](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/pubsub#readme) 、 [Spanner](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/spanner#readme) 和 [Storage](https://github.com/googleapis/google-cloud-cpp/tree/main/google/cloud/storage#readme) 的库，以及用于部署到无服务器环境的[功能框架](https://github.com/GoogleCloudPlatform/functions-framework-cpp#readme)。

我们还有许多其他的服务正在开发中，包括 Pub/Sub Lite、BigQuery 和 Firestore。我们计划不断更新 C++客户端库，以便随着 GCP 改进其现有服务或推出新服务而保持最新。

我们还致力于让 C++20 开发人员更容易使用 C++客户端库。例如，我们最近引入了对异步操作的协程支持。这可以提高复杂流媒体服务的可读性。使用协程，处理来自语音到文本服务的[异步流响应](https://github.com/GoogleCloudPlatform/cpp-samples/blob/main/speech/api/streaming_transcribe_coroutines.cc)可以是一个简单的循环:

[全代码](https://github.com/GoogleCloudPlatform/cpp-samples/blob/main/speech/api/streaming_transcribe_coroutines.cc)包括一个单独的协程来写音频

我们总是欢迎关于这些库的反馈。让我们知道你喜欢或不喜欢图书馆的什么。您可以联系您的销售代表，或者通过在我们的 GitHub 项目页面上提交[问题，或者在](https://github.com/googleapis/google-cloud-cpp/issues) [GCP 社区 Slack](https://googlecloud-community.slack.com/messages) 上加入 C++频道，直接联系工程师。我们期待您的回复。