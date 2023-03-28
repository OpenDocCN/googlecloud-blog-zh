# 如何在 GCP 上调试一个无响应的应用

> 原文：<https://medium.com/google-cloud/how-to-debug-an-unresponsive-app-on-gcp-a1c00499e2cb?source=collection_archive---------0----------------------->

**TL；博士**:你可以使用谷歌云日志和其他 GCP 工具调试一个不健康的应用程序，即使这个应用程序已经没有反应。为了解释使用 App Engine Flex 的调试过程，提供了一个测试应用程序来模拟实验中的问题。类似的调试工具和方法在其他 Google 云计算环境中广泛可用。

**背景**

谷歌云平台上的 App Engine Flex 和其他无服务器产品非常方便，让谷歌来为你管理应用。如果虚拟机…