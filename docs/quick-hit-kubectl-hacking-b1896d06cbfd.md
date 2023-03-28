# 快速命中:kubectl 黑客

> 原文：<https://medium.com/google-cloud/quick-hit-kubectl-hacking-b1896d06cbfd?source=collection_archive---------0----------------------->

我越来越喜欢一个更“流畅”的 kubectl CLI，它(更)便于将命令链接在一起。让我解释一下我的挑战。我很好奇解决这个问题的其他方法；请回复建议！

我有多个公开多个节点端口的服务:

```
kubectl get services \
--selector=component=peer,org=org1,peer=0 \
--namespace=fermium \
--context=fabric \
--output=json | **jq .items[].spec.ports[]**
{
  "name": "request",
  "nodePort": 30605,
  "port": 7051,
  "protocol": "TCP",
  "targetPort": 7051
}
{
  "name": "chaincode",
  "nodePort": 32255,
  "port": 7052,
  "protocol": "TCP",
  "targetPort": 7052
}
{
  "name": "events",
  "nodePort": 30535,
  "port": 7053,
  "protocol": "TCP",
  "targetPort": 7053
}
```

**NB** 我正在使用`--output=json`并将它传送到(优秀的)`jq`中，以保存 JSON 输出。虽然`kubectl`支持`--output=json`和`--output=jsonpath="..."`，但它返回的结果是字符串化的 Golang 类型，而不是 JSON :-(

脱离群集(！)，我想端口转发本地主机端口到服务端口，端口对端口。

对于最简单的情况，其中所有端口都属于一个 Pod，我可以使用`kubectl port-forward`，但是我希望组合来自多个服务的端口。一个有用的事实是，因为我将通过它们的`NodePorts`端口转发到这些服务，所以没有复制的可能性；节点端口在集群中是唯一的，并且在集群的节点中是一致的(我们可以选择任何节点)。

## **设置**

我正努力强迫自己在使用`kubectl`时总是指定名称空间和上下文(参见我的故事“ [Safer `gcloud `和` kubectl](/google-cloud/context-light-gcloud-and-kubectl-89185d38ce82) `”)，所以，在下面，你会看到`--namespace=${NAMESPACE}`和`--context=${CONTEXT}`。这些你都可以用，随你的便。

## 第一步:选择你的节点😃)

通常，我只获取返回的第一个节点:

```
NODE=$(\
  kubectl get nodes \
  --context=${CONTEXT} \
  --namespace=${NAMESPACE} \
  --output=jsonpath='{.items[0].metadata.name}'\
)
```

但是你可以随机选择一个:

```
NODE=$(shuf \
  --head-count=1 \
  --echo \
  $(\
    kubectl get nodes \
    --context=${CONTEXT} \
    --namespace=${NAMESPACE} \
    --output=jsonpath='{.items[*].metadata.name}'\
  )\
)
```

## 步骤 2:识别您的服务及其端口

尽管每个 Kubernetes 资源都有一个名称，但是通过类型和标签而不是名称来引用资源是一个很好的实践。例如，在创建部署和服务时，这种模式是明确需要的。这些是由将它们绑定到豆荚的选择器定义的。但是，对于所有的 Kubernetes 资源来说，这通常是一个很好的实践。这里，为了识别服务，我使用了我的服务的`component`、`org`和`peer`标签(和值):

```
kubectl get service \
--**selector=component=peer,org=org1,peer=0** \
--namespace=${NAMESPACE} \
--context=${CONTEXT} \
--output=jsonpath='{range .items[0].spec.ports[*]}{.targetPort}{.nodePort}{end}'
```

这使用了一个 jsonpath 扩展`range`来迭代`ports`数组。它输出每个`ports`的`targetPort`和`nodePort`的值。在我的例子中，每个服务的`port`与`targetPort`(Pod 的`port`)相同。

## 步骤 3:转换服务和端口映射

当我考虑这个问题时，我的想法是将`targetPort:nodePort`对的数组作为 bash 关联数组输出。代表服务端口名称的键和代表节点端口的值。

我意识到`gcloud`命令接受 `— ssh-flag`作为循环标志。我只需要为每个端口创建标志，然后将它附加到`gcloud`命令，即

```
gcloud compute instance ... --ssh-flag=... --ssh-flag=...
```

前面的`kubectl`命令的输出是所有端口和节点端口作为一个字符串的混搭。`{range}`和`{end}`之间的模板，仅输出`targetPort`和`nodePort`的值。

使用这个模板很容易生成更有用的东西:

```
{range .items[0].spec.ports[*]}
  {.targetPort}
  {":localhost:"}
  {.nodePort}
  {" "}
{end}
```

> 上面用换行符显示是为了更好地显示命令的意图。实际上，最简单的方法是将这些模板字符串保持为单行文本

对我来说，这项服务会产生:

```
7051:localhost:31833 7052:localhost:31088 7053:localhost:32085
```

> 使用字符串构造 Bash 命令并处理 Bash 的分词是很棘手的。您将看到我可以使用`Golang`模板`range`命令:`{{" — ssh-flag=\" -L"}…{" "}`为整个标志生成字符串。但是，由于 Bash 的单词拆分，这是行不通的。因此，我们需要再次迭代上面生成的端口映射列表。感谢我的同事——安德鲁——帮助我更好地理解这一点。

```
for MAPPING in $(\
  kubectl get service \
  --selector=... \
  --context=${CONTEXT} \
  --namespace=${NAMESPACE} \
  --output=jsonpath='{range .items[0].spec.ports[*]}{.targetPort}{":localhost:"}{.nodePort}{" "}{end}')
do
  FLAGS+=(--ssh-flag="-L ${MAPPING}")
done
```

我们已经解决了一个服务的问题，但是我们有多个服务。

## 4.迭代多个服务

我们已经接近我们需要的`gcloud`命令了。有用的是，因为我们现在正在生成可能附加到`gcloud`命令的字符串，我们也可以为第二个服务运行我们的命令，例如

```
kubectl get service \
  **--selector=component=orderer** \
  --namespace=${NAMESPACE} \
  --context=${CONTEXT} \
  --output=jsonpath='{range .items[0].spec.ports[*]}{.targetPort}{":localhost:"}{.nodePort}{" "}{end}'
```

> 这个命令和前一个命令的唯一区别是选择器。

倒数第二步是将每个服务的标签集提取到另一个循环中:

```
for LABELS in \
  "component=ca" \
  "component=peer,org=org1,peer=0" \
  "component=orderer"
do
  for MAPPING in $(\
    kubectl get service \
    --selector=${LABELS} \
    --context=${CONTEXT} \
    --namespace=${NAMESPACE} \
    --output=jsonpath='{range .items[0].spec.ports[*]}{.targetPort}{":localhost:"}{.nodePort}{" "}{end}'\
  )
  do
    FLAGS+=(--ssh-flag="-L ${MAPPING}")
  done
done
```

> 希望你能看到如何推广这个过程。只是标签不同的 Kubernetes 服务提供了一个简单的迭代源。

## 5.生成 gcloud 命令

因此，我们现在能够:

*   获取一个集群的`Nodes`(从中我们可以访问它的`NodePorts`)
*   枚举感兴趣的多个服务的端口详细信息
*   为 gcloud 生成合适的— `ssh-flag`语法

剩下要做的就是生成 gcloud 命令。我们需要`${NODE}`和`— ssh-flags`的名单。

```
gcloud compute ssh ${NODE} "${ARGS[A]}"
```

以下是完整的脚本:

> 我能够使用`targetPort`，因为我知道它们是不重叠的。在您的解决方案中，您可能没有这种保证。在这种情况下，您可以为自己生成本地端口号，也可以直接使用节点端口或在本地映射这些端口。

## 结论

我没有得到我想要的，但我能够生产出接近我需要的东西。理想情况下(！)，我希望能够将一个 kubectl 命令的输出通过管道传输到另一个命令的输入中，这种传输方式更像 fluent:

```
kubectl get nodes ... |
xargs shuf --head-count 1 --echo |
kubectl get services ... '{range}...{end}' |
...
```

仅此而已！