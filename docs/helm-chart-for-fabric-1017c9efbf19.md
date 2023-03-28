# 织物舵图

> 原文：<https://medium.com/google-cloud/helm-chart-for-fabric-1017c9efbf19?source=collection_archive---------0----------------------->

## 编程笔记

正如我上周在的一个[故事中提到的，我已经 **un** 成功尝试为 Kubernetes 的](/google-cloud/helm-chart-for-fabric-for-kubernetes-80408b9a3fb6) [Hyperledger Fabric](https://www.hyperledger.org/projects/fabric) 创建 [Helm](https://www.helm.sh/) 图表。GitHub 上记录了各种方法( [v1.2](https://github.com/DazWilkin/hyperledger-fabric-1.2.x) ， [v1.1](https://github.com/DazWilkin/hyperledger-fabric) )，但是没有一种方法可以在没有一些手工操作的情况下工作。这个故事试图总结我的编程笔记。

## 赫尔姆？

Helm 已经成为 Kubernetes 应用程序部署的首选，也是许多 Kubernetes 应用程序的部署选择。IIUC Helm 的下一个版本将包括重大变化，包括从集群端管理的变化。由于为 Kubernetes 构建云市场部署的需求，我几乎从一开始就使用了 Helm 的仅客户端和`helm template`方法。

我已经使用 Google 的云部署管理器、bash、Jsonnet 和 Helm 来部署 Kubernetes 解决方案，但我不相信 Helm 目前为应用程序部署开发人员提供的独特价值。因为它是事实上的标准，所以应用程序最终用户使用它会有好处。

**现有技术**

我要感谢 IBM 的各位同仁，他们在我的冒险中慷慨相助。我从 IBM 为 IBM Container Service 开发的 Fabric 的掌舵图开始这个项目。后来，IBMers 加里和 Yacov 帮助我了解了织物概念，并提供了指导。

## 读写很多

我为 Fabric 绘制的掌舵图假设有一个[读写多](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)卷可供群集使用。Google [持久盘](https://cloud.google.com/persistent-disk/)不支持读写多卷(直接)。

相反，我使用了一个 Kubernetes [样本](https://github.com/kubernetes/examples/tree/bdda4f31e60d171f77252e7d041e9c6877565f73/staging/volumes/nfs)，展示了如何在持久磁盘上运行 NFS。NFS 服务器被配置为可读写的许多永久卷和永久卷都可以对其进行声明。

对于 Fabric Helm 图表，有一个`shared`声明，用于存储 crypto-config、genesis 块、通道事务的输出，以及部署用来跟踪状态的类似信号量的文件。

这个解决方案非常有效。它有点隐蔽，但这是一个可靠的，熟悉的和廉价的方式来获得读写许多与 Kubernetes 引擎。

## 标记

Helm 有一套有价值的[最佳实践](https://docs.helm.sh/chart_best_practices)，包括资源标签。当我记录这一点时，我意识到我要么偏离了当前的指导，要么没有准确地执行它(抱歉)。

我一直试图遵循以下方法:

```
app.kubernetes.io/name: {{ include "hyperledger-fabric.chart" . }}
app.kubernetes.io/version: {{ .Chart.Version }}
app: {{ include "hyperledger-fabric.name" . }}
chart: {{ include "hyperledger-fabric.chart" . }}
release: {{ .Release.Name }}
heritage: {{ .Release.Service }}
component: peer
org: {{ $orgName }}
peer: {{ $peerID | quote }}
```

Component 反映了 Fabric 组件的类型:peer、orderer、ca 等。谨慎的做法是也使用标签来说明组织名称和对等 ID。标签可能有“太多”的地方，但这个列表远非如此。一般来说，你不应该*使用 Kubernetes 的名字来引用反映 Kubernetes 自己的方法的资源，例如，根据标签和类似的原因来选择 pod:动态性和多维性。

## 配置映射

Fabric 的配置文件`crypto-config.yaml`和`configtx.yaml`是 Kubernetes 配置映射的自然选择，两者都反映为存储库中的配置映射([链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/configtx-configmap.yaml)、[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/crypto-config-configmap.yaml))。

我面临的一个挑战是希望能够在部署期间解析这些文件以提取值。例如，`crypto-config.yaml`定义了订购者的姓名、组织及其名称以及对等用户和用户的初始数量。这些值都是构建精确网络所必需的。

然而，虽然配置映射可用于托管一组键-值对，但配置映射不允许任何值层次结构；如果值本身是键值对，则不能通过配置映射直接访问。因此，虽然将 YAML 文件编码为 ConfigMap 值很简单，但是不可能将任意的 YAML 文件反射为 config map。

与 ConfigMaps 不同，Helm 的`values.yaml`当然可以反映任意的 YAML 文件，并且 Helm 模板可以直接访问文件中的任何值。出于这个原因，我手动将`configtx.yaml`和`crypto-config.yaml`文件内容的子集反映在`values.yaml`中，这样我就可以直接访问每个组织的对等体数量。参见[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/values.yaml)。

例如，这允许:在 crypto-config.yaml 中定义的对等组织上迭代，以便创建资源:

```
{{- range $i, $org := .Values.cryptoconfig.PeerOrgs }}
{{- $orgName := $org.Name | lower }}
{{- end }}
```

**问题**:如何从头盔更直接的访问`crypto-config.yaml`和`configtx.yaml`？

**问题**:crypto-config . YAML 和 configtx.yaml 看起来有什么出入(是吗？)共享密钥已协调？为什么要分割？

**问题** : Fabric 支持 YAML 锚&引用，但是在 values.yaml 中不支持这些引用，并且大部分被展平(或排除)。

## 拔靴带

[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/bootstrap-deployment.yaml)。

在开始提供网络之前，需要做一些准备工作。这些步骤(幸运的是)是一组线性步骤，但是必须记录成功完成的步骤。完成被异步地用来阻止其他步骤的启动:例如，在`cryptogen`运行之前，不可能做任何事情(启动一个对等体)。

Kubernetes 和 Helm 都没有包含协调需要依赖和分支的部署的机制。这个事实让我顿悟到对赫尔姆的失望。我意识到，特别是它的客户端专用功能，以及`helm template`命令的名称(也许不是讽刺的),使得 Helm 更像是一个模板工具，而不是部署工具。

总之，如果 Helm 只是模板，我更喜欢使用 Jsonnet。虽然我是 Golang 的粉丝，但我不太喜欢 Golang 模板化(即使是在与 Go 一起编写时)和 for (JSON sic。)模板化，更喜欢简单的 Jsonnet。

通过 Helm，我生成了一组 Kubernetes 清单，将它们放在集群上，让集群(大部分情况下很少)协调资源的创建。

此外，因为 Helm 和 Kubernetes 都不支持依赖或分支，开发人员必须求助于模糊的方法来实现这一点。在我的例子中，我借鉴了 IBM 的解决方案对信号量文件的使用。这是一个很好的解决问题的方法，但是，如果不深入研究清单文件并想知道为什么名为`cryptogen_complete`的文件被触摸，这个意图就不明显。

因为引导步骤是一次性的，所以我使用 Kubernetes 作业资源来表示它。我经常在这个清单和其他清单中使用`initContainers`。`initContainers`，顾名思义，是主`containers`赛事的预备容器。有趣的是，`initContainers`是按照它们被定义的顺序运行的。这提供了一种方便的方法来将一组线性步骤分解成一些易于从 Kubernetes 监视中观察到的内容。

对于自举，我们:

1.  `cryptogen`生成
2.  `configtxgen`(创世纪)
3.  `configtxgen`(频道)
4.  ForEach 锚点`configtxgen`(锚点对等事务)

因此，这转化为(伪模板):

```
initContainers:# 1st step
- name: cryptogen
  image: cryptogen
  args:
  - generate# 2nd step
- name: configtxgen-genesis
  image: configtxgen
  args:
  - ...
  - -outputBlock# 3rd step
- name: configtxgen-channel
  image: configtxgen
  args:
  - ...
  - outputCreateChannelTx# 4th step
{{- range $org := .Values.configtx.Organizations }}
- name: anchor-{{ $org.ID | lower }}
  image: configtxgen
  args:
  - ...
  - -outputAnchorPeersUpdate
{{- end }}containers:
...
```

需要注意几件事。该作业中的实际`containers`主要是输出步骤，动作在`initContainers`中。我为`cryptogen`和`configtxgen`构建了容器。这些二进制文件是为 Debian|Ubuntu(使用 libc)构建的，我想使用一个轻量级的运行时环境，比如 Alpine。将这些二进制文件放入容器中增加了流程的清晰度，并且与作为容器映像提供的其他 Fabric 二进制文件更加一致。

容器执行`configtxgen` `inspectBlock`和`inspectChannelCreateTx`(查看)命令。还有一个容器触动`/shared/bootstrapped`来充当自举成功完成的信号量。

## 渠道创建和对等加入

[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/channel-create-job.yaml)，[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/peer-join-deployment.yaml)

创建渠道是 Kubernetes 的另一项工作。与其他 Fabric 资源一样，它使用一个`initContainer`来检查 semamphore 文件的创建，从而阻止成功完成自举。最初，这项工作还阻止了使用`peer node status`的锚节点的可用性，但是 Yacov@IBM 指出这是多余的，并且还会导致部署问题。一旦成功创建了通道，就会创建另一个文件信号量，它发出信号通知对等体加入命令继续执行。

在这个图表中，每个对等体然后加入通道。因此，部署类似于对等体创建，它遍历组织，然后动态地遍历组织中定义的每个对等体，并运行正确配置的`peer channel join`命令。

然后，每个组织必须用该组织的锚定对等点更新一次渠道。

## CA，CLI，订购者

[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/ca-deployment.yaml)、[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/cli-deployment.yaml)、[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/orderer-deployment.yaml)

这些资源易于部署。它们采用跨结构节点类型使用的简单模式。单个 Pod(尽管是由部署管理的)由一个服务作为前端。该服务为结构节点和端口提供 DNS 名称，例如`7050`上的 gRPC。

CLI 是调试、安装和实例化链代码的有用工具。当我在解决这个命令失败的原因时，我将命令模板化以使自己快速回到 instantiate 命令。

进入 CLI:

```
kubectl exec \
--stdin \
--tty \
$(\
  kubectl get pods \
  --selector=**component=cli,org=org1** \
  --output=jsonpath="{ .items[0].metadata.name }" \
  --namespace=${NAMESPACE} \
  --context=${CONTEXT} \
) \
--container=cli \
--namespace=${NAMESPACE} \
--context=${CONTEXT} \
-- bash
```

然后:

```
peer chaincode install \
--name=${NAME} \
--version=${VERSION} \
--path=github.com/chaincode/example02/go/2018-08-24 [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2018-08-24 [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2018-08-24 [main] main -> INFO 003 Exiting.....
```

这可以通过以下方式确认:

```
peer chaincode list \
--channelID=$CHANNEL_NAME \
--installedGet installed chaincodes on peer:
Name: ex02, Version: 2.0, Path: github.com/chaincode/example02/go/, Id: 33620f48ee049ffbb7f763f77a3bde54b768ebba9d502532c9bbb98e66fdba3e
2018-08-24 [main] main -> INFO 001 Exiting.....
```

挑战在于实例化:

```
peer chaincode instantiate \
--orderer=${RELEASE_NAME}-hyperledger-fabric-orderer:7050 \
--cafile=/shared/.../tlsca.example.com-cert.pem \
--channelID=$CHANNEL_NAME \
--name=${NAME} \
--version=${VERSION} \
--ctor='{"Args":["init","a", "100", "b","200"]}' \
--policy="OR ('Org1MSP.peer','Org2MSP.peer')"
```

其价值与以下各项保持一致:

```
Error: Error endorsing chaincode: rpc error: code = Unknown desc = timeout expired while starting chaincode ex02:2.0(networkid:dev,peerid:org1-peer0,tx:82221c06970c59e6a745c04515446d6776aa2f8899fc1d2a808954c81db07ee1)
```

感谢 Yacov@IBM，这个问题被确定为使用 docker-in-docker 的 Fabric 的挑战，我在下面的“**Peer**”中简要总结了这个问题，并在我上周的总结故事[ [链接](/google-cloud/helm-chart-for-fabric-for-kubernetes-80408b9a3fb6) ]中更完整地总结了这个问题。

## 同行

[链接](https://github.com/DazWilkin/hyperledger-fabric-1.2.x/blob/master/templates/peer-deployment.yaml)

成功完成引导步骤时的对等创建块。其他几个部署步骤同样会阻止引导。为了明确这一事实，我使用了`initContainers`，他们简单地说:

```
- name: await-bootstrapped
        image: busybox
        imagePullPolicy: IfNotPresent
        command:
        - ash
        - -c
        - |
          while [ ! -f /shared/bootstrapped ]; do
            echo Awaiting /shared/bootstrapped
            sleep 15s
          done
```

我想根据`crypto- config.yaml`中定义的意图提供对等点。正如我上面提到的，我从`crypto-config.yaml`和`configtx.yaml`中复制了显著的组块来达到这个目的。因此，对等部署包括对[对等]组织的迭代，以及对每个对等组织的迭代:

```
{{- range $i, $org := .Values.cryptoconfig.PeerOrgs }}
  {{- $orgName := $org.Name | lower }}
  {{- range $j, $peerID := until ( $org.Template.Count | int) }}
    {{- $orgFullName  := printf "%s.example.com" $orgName }}
    {{- $peerName     := printf "peer%d" $peerID }}
    {{- $peerFullName := printf "%s.%s" $peerName $orgFullName }}
    ...
  {{- end }}
{{- end }}
```

在内部循环中，为每个对等体和将对等体(Pod)暴露给结构网络中的其他实体的伴随服务创建部署。

取一个组织名称(例如原型`Org1`)和一个对等名称(例如原型`Peer0`)并将它们映射到`x-hyperledger-fabric-org1-peer0`和`org1.example.com`是很简单的。我将把这个讨论推迟到“**命名**”部分(见下文)。

同行也挂载`docker.sock`。当调用 chaincode `instantiate`命令时，对等体获取`install` chaincode 并使用 [docker-in-docker](https://blog.docker.com/2013/09/docker-can-now-run-within-docker/) 到`create`一个 docker 映像，`attach`然后`start`它:

```
IMAGE=$(\
  docker images --format="{{ .Repository }}" \
  | grep dev-org1-peer0-example02-2.0\
)docker events --filter=image=${IMAGE}2018-08-22 container **create** 132bbe(image=dev-org1-peer0-ex02-2.0)
2018-08-22 container **attach** 132bbe(image=dev-org1-peer0-ex02-2.0)
2018-08-22 container **start** 132bbe(image=dev-org1-peer0-ex02-2.0)
```

但是，这会导致容器在主机上的 Docker 引擎的上下文中运行(在本例中是 Kubernetes 节点)。如果容器试图与例如`x-hyperledger-fabric-org1-peer0`对话，它会失败。该名称是一个 Kubernetes DNS 名称，Kubernetes 名称在群集外部无法解析。这是我上周在总结[故事](/google-cloud/helm-chart-for-fabric-for-kubernetes-80408b9a3fb6)中描述的突出问题之一。

**调试:**如何找到一个 Pod 正在运行的节点？

```
PROJECT=[[YOUR-PROJECT]]
NAMESPACE=[[YOUR-NAMESPACE]] # Optional
CONTEXT=[[YOUR-CONTEXT]] # Optional
SELECTOR="**component=peer,org=org1,peer=0**"gcloud compute ssh $(\
  kubectl get pods \
  --selector=${SELECTOR} \
  --output=jsonpath="{.items[0].spec.nodeName }" \
  --namespace=${NAMESPACE} \
  --context=${CONTEXT}) \
--project=${PROJECT}
```

> **NB** 这结合了一个`kubectl`命令，该命令获取一个 Pod 的节点名，然后与`gcloud`一起使用，ssh 到实例中。

我仍然不清楚哪些环境变量是必需的，哪些是可选的，哪些环境变量的默认值对于简单的`peer node start`命令来说是可以接受的。

特性请求:我希望文档能够扩展到包含与每个命令相关的环境变量列表。

## 命名

命名仍然是一个挑战。协调 Helm|Kubernetes 命名(最佳实践)和 Fabric 命名的问题仍未解决。

## 港口

最初，我天真地为每个服务生成了`NodePorts`。如果没有指定`--type=NodePort`服务，则分配一个可用端口。最初，我没有仔细考虑这个过程，并试图计算服务的节点端口，并将它们分布在端口空间中。

这是个坏主意。

这不仅使我有可能需要调整端口空间来确保必要的组织、对等体等。有足够的港口满足他们的需求，但更重要的是，我忘记了 Kubernetes 的一个宗旨。也就是说，我总是可以查找分配给服务的节点端口:

```
kubectl get services \
--selector=${SELECTOR} \
--output=jsonpath='{.spec.ports[?(@.name=="grpc")].nodePort}" \
--namespace=${NAMESPACE} \
--context=${CONTEXT}
```

> **NB** 上面用 JSONPath 对`ports`的数组进行过滤，得到一个名为`grpc`的，它的`nodePort`值就是结果。

但是，经过思考，我意识到大多数服务不需要在原型中的集群之外公开，因此，我可以完全放弃节点端口，使用集群 IP。

当我需要向外部公开结构节点时，我将让集群自动分配节点端口，然后使用类似上面的查询来确定特定资源的端口。

## 日志

日志记录非常重要，使用 Kubernetes 引擎可以很容易地从任何 Pod |容器中快速获取日志。再一次，我在调试过程中反复检查日志，并从模板化命令中受益:

```
PROJECT=
CLUSTER=
REGION=
NAMESPACE=
CONTEXT=COMPONENT="peer"
ORG="org1"
PEER="2"POD=$(\
  kubectl get pods \
  --selector=component=${COMPONENT},org=${ORG},peer=${PEER} \
  --namespace=${NAMESPACE} \
  --context=${CONTEXT} \
  --output=jsonpath="{.items[].metadata.name}"\
) && echo ${POD}AFTER=$(date --rfc-3339=s --date="1 hour ago" | sed "s| |T|")
BEFORE=$(date --rfc-3339=s | sed "s| |T|")FILTER="resource.type=\"k8s_container\" "\
"resource.labels.location=\"${REGION}\" "\
"resource.labels.cluster_name=\"${CLUSTER}\" "\
"resource.labels.pod_name=\"${POD}\" "\
"timestamp>=\"${AFTER}\" "\
"timestamp<=\"${BEFORE}\""gcloud logging read "${FILTER}" \
--project=${PROJECT} \
--format=json \
--order=asc \
| jq --raw-output '.[].textPayload | rtrimstr("\n")'
```

所以，唷！但我向你保证，这很有帮助。-)

上半部分仅仅定义了常量。再一次，Kubernetes 广泛使用标签，确实有助于过滤资源。在这里，我通过各种对等节点进行过滤。`kubectl`用于识别感兴趣的 Pod 的运行时名称。

然后，这被合并到提供给`gcloud logging`的冗长的过滤字符串中。本质上，我们提供了集群的规范(名称、区域)、Pod 的名称，并且在本例中，定义了覆盖最后一个小时的时间戳。这可以用`--freshness`标志来代替，但是它只支持降序排列(这不太有用)。

最后，我们把所有东西放在一起，根据`JSON`中的过滤器提取日志，然后通过管道传输到`jq`，提取 textPayload，并修剪多余的换行符(通过 Fabric？)在日志中:

```
[nodeCmd] serve -> INFO 001 Starting peer:
 Version: 1.1.0
 Go version: go1.9.2
 OS/Arch: linux/amd64
 Experimental features: false
 Chaincode:
  Base Image Version: 0.4.6
  Base Docker Namespace: hyperledger
  Base Docker Label: org.hyperledger.fabric
  Docker Namespace: hyperledger
[ledgermgmt] initialize -> INFO 002 Initializing ledger mgmt
[kvledger] NewProvider -> INFO 003 Initializing ledger provider
[kvledger] NewProvider -> INFO 004 ledger provider Initialized
[ledgermgmt] initialize -> INFO 005 ledger mgmt initialized
[peer] func1 -> INFO 006 Auto-detected peer address: 10.0.2.8:7051
[peer] func1 -> INFO 007 Returning x-hyperledger-fabric-org1-peer0:7051
[peer] func1 -> INFO 008 Auto-detected peer address: 10.0.2.8:7051
[peer] func1 -> INFO 009 Returning x-hyperledger-fabric-org1-peer0:7051
[eventhub_producer] start -> INFO 00a\ Event processor started
```

## 结论

当我在我的编程笔记中回顾它时，我将继续用额外的信息调整这些笔记。希望这对别人有帮助|有见地。

仅此而已！