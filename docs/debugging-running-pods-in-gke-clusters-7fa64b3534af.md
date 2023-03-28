# 调试 GKE 集群中的运行窗格

> 原文：<https://medium.com/google-cloud/debugging-running-pods-in-gke-clusters-7fa64b3534af?source=collection_archive---------1----------------------->

一段时间以来，Kubernetes 支持[短暂容器](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)。从 Kubernetes 版本 1.18 开始，除了一大套其他的[故障排除方法](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application)之外，短命吊舱可以用于[调试运行吊舱](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container)。虽然 GKE 已经支持 Kubernetes 1.18，但`kubectl debug`命令仍然不可用。主要是因为这个特性在 Kubernetes API 中仍然标记为 *Alpha* 。那么，除了[检查 GKE 应用程序日志和跟踪](https://cloud.google.com/blog/products/containers-kubernetes/tools-for-debugging-apps-on-google-kubernetes-engine)之外，您还能做什么呢？

可以从托管虚拟机访问正在运行的 pod 的容器。在 GKE，大多数集群使用 [COS](https://cloud.google.com/kubernetes-engine/docs/concepts/node-images#cos) 来运行工作节点。当您 SSH 到节点时，您仍然缺少 root 访问权限以及许多调试所需的有用实用程序。在 COS 中，您可以使用 [COS 工具箱](https://cloud.google.com/container-optimized-os/docs/how-to/toolbox)来调试您的运行吊舱。工具箱最初是为了调试节点问题而创建的，但是可以很容易地转换为运行 pod 调试工具。例如，如果您需要捕获来自 pod 的流量，请执行以下操作:

1.  SSH 到 pod 运行的节点(使用`kubectl get po -o wide`查看节点名称)。
2.  运行[工具箱](https://cloud.google.com/container-optimized-os/docs/how-to/toolbox)。
3.  安装并运行 tcpdump 以捕获来源等于 pod IP 的所有数据包。
4.  将转储从节点复制到您的工作站。