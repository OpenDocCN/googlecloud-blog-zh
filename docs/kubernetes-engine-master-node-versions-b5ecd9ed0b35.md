# Kubernetes 引擎主|节点版本

> 原文：<https://medium.com/google-cloud/kubernetes-engine-master-node-versions-b5ecd9ed0b35?source=collection_archive---------0----------------------->

## 鲁布·戈德堡解决方案

> **更新 18–04–25**:末尾增加了一些 Golang 和一些 Python。

我是 pernickety，当我创建 Kubernetes 引擎集群时，我生活在边缘，使用最新的|最棒的主和节点版本。Kubernetes 引擎[默认](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create#--cluster-version)为‘服务器指定’(`defaultClusterVersion`)版本，通常不是最新的。

然而，在创建集群时，我一直在观察作为`cluster-version`值提供的版本，因为直到最近，我还不知道如何以编程方式枚举各种可能性:

```
gcloud container get-server-config \
--region=$REGION \
--project=$PROJECT \
--format="json"
Fetching server config for us-west1
{
  "defaultClusterVersion": "1.8.8-gke.0",
  "defaultImageType": "COS",
  "validImageTypes": [
    "COS",
    "UBUNTU"
  ],
  "validMasterVersions": [
    "1.9.6-gke.1",
    "1.9.6-gke.0",
    "1.9.3-gke.0",
    "1.8.10-gke.0",
    "1.8.8-gke.0"
  ],
  "validNodeVersions": [
    "1.9.6-gke.1",
    "1.9.6-gke.0",
    "1.9.3-gke.0",
    "1.9.2-gke.1",
    "1.8.10-gke.0",
    "1.8.8-gke.0",
    "1.8.7-gke.1"
  ]
}
```

> **NB** 我假设(！)主版本和节点版本同步。到今天为止，最新最好的版本应该是“1.9.6-gke.1”

那么，我该如何获取这个值呢？

一个假设是最新版本总是这些列表的第 0 个元素:

```
gcloud container get-server-config \
--region=$REGION \
--project=$PROJECT \
--format="value(validMasterVersions[0])"
Fetching server config for us-west1
1.9.6-gke.1
```

但是，这有什么意思呢？-)

[jq](https://stedolan.github.io/jq/) 来救援…

## 说明

我们运行和以前一样的`gcloud container get-server-config`命令。这一次，我们将输出格式化为`JSON`并通过管道传输到 jq。

为了使它稍微清楚一点，定义了两个函数`to_gke_semver`(第 7+8 行)和`from_gke_semver`(第 9+10 行)。从上面的命令输出中，您会看到它产生了形式为`1.9.6-gke.1`的字符串。这就像一个扩展的 semver。我们希望能够对这种类型的字符串进行排序。因此，`to_gke_semver`将该字符串转换成 JSON `{“major”:”1", “minor”:”9", “patch”:”6", “gke”:”1"}`，然后我们将使用它作为排序的基础。函数`from_gke_semver`将 JSON 转换回字符串。

我们使用 jq 的`reduce`函数(第 11 行)，它将一个数组作为输入，一个累加器，并依次对每个数组项应用一些函数，通常将结果应用到累加器。在我们的例子中，累加器最初是最低值的 GKE·塞弗`{“major”:”0", “minor”:”0"….}`(第 14-19 行)。我们的函数将已经转换成 JSON semver 对象的`validMasterVersion`(第 12 行)与累加器进行比较，如果累加器更大，就用 item 替换累加器。

较大的值具有较大的`major`值。对于具有相同`major`值的两个 semvers，较大的具有较大的`minor`值，等等。等等。

最后，我们将最大的`validMasterVerson`作为 JSON 对象，并将其转换回字符串(第 36 行)。这就是`${LATEST}`的价值。

将上面的脚本片段复制并粘贴到您的 shell 中。如果像我一样，今天，你会得到:

```
echo ${LATEST}
1.9.6-gke.1
```

然后，您可以:

```
gcloud beta container clusters create $CLUSTER \
--username="" \
--cluster-version=${LATEST} \
--machine-type=custom-1-4096 \
--image-type=COS \
--num-nodes=1 \
--enable-autorepair \
--enable-autoscaling \
--enable-autoupgrade \
--enable-cloud-logging \
--enable-cloud-monitoring \
--min-nodes=1 \
--max-nodes=2 \
--region=$REGION \
--project=$PROJECT \
--preemptible \
--scopes="[https://www.googleapis.com/auth/cloud-platform](https://www.googleapis.com/auth/cloud-platform)"
```

## 结论

是的，它看起来过于复杂，但是有更好的方法来确定这个值吗？

## 计算机编程语言

感谢 Sean 跳出我的思维定势，提供了 Python 替代方案:

然后，与`jq:`一样

```
LATEST=$(\
  gcloud container get-server-config \
  --region=$REGION \
  --project=$PROJECT \
  --format="json" \
  | python3 python.py) \
&& echo ${LATEST}
```

## 戈朗

然后，和`jq:`一样

```
LATEST=$(\
  gcloud container get-server-config \
  --region=$REGION \
  --project=$PROJECT \
  --format="json" \
  | go run main.go) \
&& echo ${LATEST}
```

还是`go build`吧。