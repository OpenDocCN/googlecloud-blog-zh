# 用 kubectl 以文本格式显示 GKE 日志

> 原文：<https://medium.com/google-cloud/display-gke-logs-in-a-text-format-with-kubectl-db0169be0282?source=collection_archive---------0----------------------->

如果使用 GKE 和云日志进行日志监控，则需要将日志打印为 JSON 格式，以便在云控制台上使用更有效的监控。然而，如果您在 CLI 上使用 **kubectl logs** 来查看日志，日志将无法读取。它将每一行打印为一个 JSON 对象。

大多数情况下，最好使用 Google Cloud Console 来过滤日志并显示详细信息，但有时您希望在 CLI 上快速查看日志。

为了以文本格式查看基于 JSON 的日志，我编写了一个简单的 shell 脚本。基本上，这个脚本使用 **jq** 将 JSON 对象转换成文本。你可以在 Github 上访问脚本和安装细节:【https://github.com/berker/kube-gke-logs 