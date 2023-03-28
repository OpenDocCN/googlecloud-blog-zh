# cygwin 上的 Google Cloud SDK(g Cloud)——差不多…

> 原文：<https://medium.com/google-cloud/google-cloud-sdk-gcloud-on-cygwin-almost-9be91e0006fb?source=collection_archive---------3----------------------->

我在 windows 机器上工作，通过 windows installer 安装 [Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstart-windows) 。到目前为止，我能够在我的 cygwin 环境中处理几乎所有的事情，除了几个例外。

在对 [GKE](https://cloud.google.com/container-engine/) 集群进行身份验证后，kubectl 的“cmd-path”配置存在一些问题:

```
Unable to connect to the server: error executing access token command "/cygdrive/c/Users/brett/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud config config-helper --format=json": err=exec: "/cygdrive/c/Users/brett/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud": file does not exist output=
```

为了在每次成功收集端点和 auth 数据后解决这个问题，我需要更改~/中的 kubeconfig 条目。kube/配置:

```
- name: gke_ops-iac-sb_us-east1-b_iac-test-cluster
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /cygdrive/c/Users/brett/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

你可以看到 cmd 路径指向我的 windows 安装版本的 gcloud。(注意:我已经在不同的目录中不加空格地试过了，甚至把它指向 cygwin 下的一个位置路径，结果都是一样的。)这在使用 kubectl 做任何事情时都会抛出一个错误:

```
$ kubectl cluster-info
Unable to connect to the server: error executing access token command "/cygdrive/c/Users/brett/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud config config-helper --format=json": err=exec: "/cygdrive/c/Users/brett/AppData/Local/Google/Cloud SDK/google-cloud-sdk/bin/gcloud": file does not exist output=
```

为了解决这个问题，我将 cmd 路径改为:

```
cmd-path: gcloud
```

问题是，每次认证都必须这样做。我还尝试过使用 curl 将 cloud sdk 直接安装到 cygwin 上。这工作，但是我无法安装 kubectl 后:

```
$ gcloud components install kubectl
ERROR: (gcloud.components.install) The following components are unknown [kubectl].
```

我也无法连接到豆荚:

```
$ kubectl.exe exec -it my-pod --namespace=my-namespace -- /bin/bash
Unable to use a TTY - input is not a terminal or the right kind of file
```

我遇到的下一个问题是关于将 docker 图像推送到[谷歌容器注册中心](https://cloud.google.com/container-registry)。

```
$ gcloud docker -- push gcr.io/ops-iac-sb/gcloud-slave:v1
The push refers to a repository [gcr.io/ops-iac-sb/gcloud-slave]
f2c2cdf1f2e9: Preparing
b5b8df85aa3c: Preparing
24b3e47367c1: Preparing
a75caa09eb1f: Preparing
9b790cee2175: Waiting
348667646bbd: Waiting

denied: Unable to access the repository; please check that you have permission to access it.
```

我还没有找到解决这个问题的方法。我必须从 windows Google Cloud SDK shell 运行这些命令。

我喜欢能够完全使用来自 cygwin 的 Google Cloud SDK，所以我会继续探讨这个问题。如果有人有任何想法或者比我更成功，请告诉我！