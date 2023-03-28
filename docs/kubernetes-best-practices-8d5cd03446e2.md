# Kubernetes 最佳实践

> 原文：<https://medium.com/google-cloud/kubernetes-best-practices-8d5cd03446e2?source=collection_archive---------1----------------------->

我和一位前谷歌员工 SRE 聊天，他(正确地)指出 Kubernetes 发展非常快(太快以至于不能保持流行)，使用(许多)新颖的概念，并且有(太多)方法来解决同样的问题。

这大部分是真的，不一定是坏事，也不一定与任何其他技术不同。我不同意的是，这些因素阻碍了他采用 Kubernetes。我会鼓励你投入进去。Kubernetes 是成功的，尽管有这些(合理的)担忧，因为它非常非常好。

在本帖中，我将向您提供一些基本的最佳实践，希望能帮助您抓住这一技术的容器并深入其中。

排名不分先后:

1.  **让别人去做苦工吧！**

使用 Kubernetes 服务，如 Kubernetes 引擎。除非你很好奇，你是一个开发 Kubernetes 的开发者，或者你是一个平台提供商，有客户要求 Kubernetes 服务，省掉麻烦，使用 Kubernetes 服务。你建造了自己的家和汽车吗？或者你真的喜欢睡在一个狼不会吹倒的地方，开着一辆可靠地把你从 A 带到 B 的车？

所以，如果你读过我的其他帖子，我也建议你评估一下[区域集群](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-zone-and-regional-clusters#regional)，这样你就可以看到这样的东西:

```
gcloud beta container clusters create ${CLUSTER} ...
gcloud beta container clusters get-credentials ${CLUSTER} ...
```

然后，你就可以开始了:

```
kubectl apply --filename=marvels.yaml
```

**2。试想想“Kubernetes”**

与其他平台相比，这对于 Kubernetes 引擎来说可能是一个更大的挑战，但是在 Google 云平台上，您必须了解 Kubernetes 中资源的状态(例如节点、入口)，同时还要了解计算引擎中的底层资源(例如虚拟机、HTTP/S 负载平衡器)。这个粒子-波二元性问题是不幸的。感谢戴尔·h 首先为我阐明了这一点。

在可能的情况下，试着坚持从 Kubernetes 的资源的角度思考，而忽略潜在的 GCE 资源。现在，我已经花了一年多的时间将我的工作偏向于 Kubernetes，这使得纯粹从服务(和入口)公开的“一定数量”的 pod 的角度来考虑变得更加容易。

**3。名称空间，名称空间，名称空间**

> U **pdate** :感谢 [Michael Hausenblas](https://medium.com/u/5b086b7c6aea?source=post_page-----8d5cd03446e2--------------------------------) 教给我一个最佳实践，即不要在 Kubernetes YAML 文件中引用名称空间。虽然你应该总是使用名称空间，但在你`apply`文件时指定这些名称空间提供了更多的灵活性和对不同的场景使用相同的 YAML 文件的能力。看迈克尔的文章[这里](https://blog.openshift.com/kubernetes-application-operator-basics/)。

几个月前，Mike Altarace 和我在博客上讨论了 Kubernetes 中的名称空间以及应该在哪里使用它们。从那时起，我几乎忽略了我自己的建议，认为我的用例是如此之小，以至于使用名称空间会过于紧张。我错了。始终使用名称空间。

就像容器对于流程一样，名称空间对于 Kubernetes 项目也是如此。除了名称空间传达的安全边界之外，它们是划分您的工作的一种极好的方式，并且它们产生了重置或删除它的一种极好的方式:

```
kubectl delete namespace/$WORKING_PROJECT
```

唯一的缺点是，当使用非`default`名称空间时，您需要在`kubectl`命令上指定您的工作名称空间`--namespace=$WORKING_PROJECT`，在我看来，这是一个很好的保护措施。然而，你可以总是`--all-namespaces`或者设置一个不同的名称空间作为你的默认名称空间([链接](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference))。

**4。解决问题的方法太多**

这是一个令人沮丧的担忧。我认为这可能是不真实的，但是，如果没有好的指导和最佳实践，也许看起来有太多相似的方法来解决同一个问题。我有一个常用的模式，我在这里总结一下，以鼓励讨论:

*   YAML 文件是冷藏的知识(见#5)
*   你的容器应该做好一件事
*   始终部署部署(参见第 6 条)
*   如果您想要 L7(也称为 HTTP/S)负载平衡，请使用 Ingress(参见#7 — ha！)
*   使用机密安全地管理凭证([链接](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/))

**5。偏向** `**kubectl apply --filename**` **而不是备选方案**

使用`kubectl create namespace/${WORKING_DIR}`很容易获得快速响应，但是，在几个这样的命令之后，您可能想知道您是如何到达当前状态的——更重要的是——如何重新创建这个状态。我鼓励你创建 YAML 文件来描述你的资源，而不是等效的`kub ectl create`命令。

我鼓励您熟悉*优秀*的 Kubernetes API 文档([链接](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)、[链接](https://kubernetes.io/docs/reference/)和 [1.10](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/) )，这些文档详尽、准确且易于浏览(也许是最完美的文档！？).但是，即使有了这个强大的工具，有时将一个为您工作的`kubectl`命令转换成 YAML 还是有点困难。它不是:

```
kubectl get deployment/${MY_DEPLOYMENT} --output=yaml
kubectl get service/${MY_SERVICE} --output=yaml
kubectl get anything/${MY_ANYTHING} --output=yaml
```

如果您愿意，可以将结果通过管道传输到一个文件中，但是要将这些结果作为等价(！idspnonenote)的基础。)YAML 文件。您需要删除任何实例引用。

一旦你完成了`masterpiece.yaml`，我鼓励你总是`apply`做最初的创建，`apply`做任何后续的更新，如果你必须的话`delete`。就是这样！

```
kubectl apply --filename=masterpiece.yaml
kubectl delete --filename=masterpiece.yaml
```

*小见识*:你不需要为了部署而把 YAML 文件拉到本地。您也可以为`kubectl apply --filename`提供 URL，只要任何依赖文件都是本地引用，部署就会工作。

*小见识*:我只在 Kubernetes-land 看到过这种用法，但这是一种合法的做法，你可以用`---`和`...`文件分隔符将多个 YAMLs 文件合并成一个 YAML 文件。因此，与命名空间 YAML、部署 YAML 和服务 YAML 不同，您可能有一个将所有三个文件合并为一个的大型 YAML。

这是有效的 YAML(将其复制并粘贴到例如 [YAML 林特](http://www.yamllint.com/))。这是无效的，因为每个规范都是不完整的，但是如果每个规范都是完整的，这是非常好的 YAML，也是保持资源关联的好方法。

一个批评见#B (Xsonnet)。

6。使用部署

在[部署](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)中有很多力量，我的指导是:始终使用部署，每时每刻。即使你正在部署你的第一个单人吊舱`nginx`。部署是“头等舱”旅行，以蔻驰的价格来说，你可以进入滚动部署，你犯了一个错误，重新`apply`和 Kubernetes 负责杀死顽皮的豆荚，并用表现良好的豆荚替换它们。

**7。负载平衡器和入口**

这些造成了混乱。在我看来(我可能错了)，当我使用`--type=LoadBalancer`创建服务时，我想要|得到一个网络 LB。如果我想要 HTTP/S(第 7 级)负载平衡，我需要创建一个入口。Ingress 是一个令人困惑的 Kubernetes 资源。简单来说，L7==Ingress(结果是大量的配置能力)。

Kubernetes 引擎将入口资源显示为 GCE HTTP/S 负载平衡器。Christopher Grant 在他的文章中很好地揭开了入口的神秘面纱(这里的和这里的)。

**8。节点端口**

我没有(从来没有？)直接创建了一个 ClusterIP。我对 Kubernetes 做了一些事情，导致服务被 ClusterIPs 公开。主要是(！)我创建了节点端口，或者我做了一些使用节点端口的事情(例如创建入口资源)。

节点端口与 Kubernetes 节点相关联，它们是端口。它们提供的强大功能是集群中的每个节点(或者是这个节点池？`[[TODO]]`)在同一个(节点)端口上公开同一个服务。

如果我创建一个在节点端口`X`上公开的服务，我可以保证，如果我在集群中的*任何*节点上访问该端口，我将访问该服务。这构成了 Kubernetes 负载平衡功能的基础，因为集群能够将传入的服务请求路由到任何节点上的这个端口。

Google Cloud SDK(又名`gcloud`)包括一个`ssh`客户端，可以轻松连接到计算引擎虚拟机(您应该记得是 Kubernetes 集群节点)。`ssh`客户端包括端口转发功能。因此，如果我们想要连接到一个 Kubernetes 服务，并且我们可以查找该服务的节点端口，那么我们可以很容易地(！)通过端口转发(使用`gcloud`或任何`ssh`客户端)到任何节点上端口，端口转发到服务。

下面的例子使用`kubectl`来抓取集群中的第 0 个节点。Kubernetes 引擎节点名称与计算引擎虚拟机名称相同。给定一个在名为`${MY_NAMESPACE}`的名称空间中名为`${MY_SERVICE}`的服务，我们确定服务的节点端口。然后，我们切换到`gcloud`，并使用其内置的`ssh`进行端口转发(使用`--ssh-flag="-L XXXX:localhost:XXXX`)。

```
NODE_HOST=$(\
  kubectl get nodes \
  --output=jsonpath="{.items[0].metadata.name}")NODE_PORT=$(\
  kubectl get services/${MY_SERVICE} \
  --namespace=${MY_NAMESPACE} \
  --output=jsonpath="{.spec.ports[0].nodePort}")echo ${NODE_PORT}gcloud compute ssh ${NODE_HOST} \
--ssh-flag="-L ${NODE_PORT}:localhost:${NODE_PORT}" \
--project=${YOUR_PROJECT}
```

这有什么厉害的？现在，您可以像在本地一样访问服务，而不必在防火墙上打孔。

节点端口是高编号端口(~30，000–32，767)。

**9。黑客** `**kubectl**` **使用 JSON**

谷歌的[云 SDK](https://cloud.google.com/sdk/docs/) (又名`gcloud`)确实很优秀但是`kubectl` (Kubernetes CLI)更好(原文如此)。一个强大的功能是格式化和过滤输出。这允许以非编码(非 API 争论)的方式使用来自 Kubernetes 集群的信息来扩展脚本和其他工具。

所有的 Kubernetes 资源状态都可以通过例如`kubectl get`(根据我的经验，它比`kubectl describe`)命令来访问。然后，剩下的就是在 JSON 的大海里捞针了。

诀窍是:

```
kubectl get [resource]/[resource-name] --output=JSON
```

然后查看结果，开始构建查询字符串:

```
kubectl get [resource]/[resource-name] --output=jsonpath=".items[*]"
```

并反复细化结果集，直到得到您所寻找的项目。这里有一个适用于任何集群的示例:

```
kubectl get nodes --output=json
kubectl get nodes --output=jsonpath="{.items[*]}
kubectl get nodes --output=jsonpath="{.items[0]}
kubectl get nodes --output=jsonpath="{.items[0].metadata.name}
```

最后，学习一种 JSON 解析工具并将该工具应用于所有 JSON 解析需求，这是一个很好的理由(也是*nix 的宗旨)。在这种情况下，`jq`有没有合理的竞争对手？我想不会。

加上 [jq](https://stedolan.github.io/jq/) 有一个优秀的游乐场(jqplay.org)。

**答:使用标签**

这是一个漫长的过程，但是现在所有的软件服务都支持任意标记资源的概念(通常是键值对)。其强大的原因在于，这种元数据提供了一种开放式的、完全由用户定义的资源查询方式。Kubernetes 固有地使用这个原则；这是一种内在的能力，而不是一种事后想出来的|附加功能。

Kubernetes 服务公开任意数量的 Kubernetes Pods。服务不会公开名为“Henry”的 pod 或包含副本集的 pod。相反，服务公开其标签符合服务规范中定义的标准的 pod，这些标签当然是用户定义的。

**注意**在上面的例子中，我们使用了一个名为`project-x`的名称空间，这个名称空间规范出现在`Namespace`(当它被创建时)、`Deployment`中以定义部署存在的位置，并且出现在`Service`中。`Deployment`(称为`microservice-y`)将创建一个副本集(此处隐式指定；这是部署所创造的)将维持 100 个吊舱。每个 Pod 将有一个标签`app: publicname-a`，并包含一个基于名为`image-grpc-proxy`的图像的名为`grpc-proxy`的容器。这个服务叫做`service-p`。最重要的是，该服务选择(！)具有标签`app: publicname-a`的 pod(仅在`project-x`名称空间中)。该服务将选择具有该标签(键:值)对的任何 Pod(在`project-x`名称空间中),而不仅仅是在该部署中创建的那些 Pod。该服务不通过 pod 的名称(基于部署名称)、其容器名称或容器图像名称来引用 pod，只引用与 pod 相关联的标签。

这不是一个好的做法，但它证明了这一点。如果您要运行一个类似于上面的配置，然后单独创建一个运行 Nginx 的 Pod(在`project-x`名称空间中),然后您给它添加标签`app: publicname-a`,它将很快被合并到由`service-p`服务聚合的 Pod 集合中。如果您要删除由服务聚合的任何 Pod 的标签，该 Pod 将停止被包含。

此功能通过滚动更新来体现，其中部署创建新的副本集，该副本集包括版本 X '的新单元，不同于版本 X 的副本集和单元。服务可被定义为展示跨这两个版本运行的单元的交集，因为它不是根据副本集或特定单元来定义的，而是通过用户定义的标签(“选择器”)来定义的，这些标签是在创建单元时由您应用的。

那是很厉害的！

b .使用 Jsonnet，可能是 Ksonnet

两个挑战同所有(！？)结构化格式(YAML、JSON、XML、CSV-)是自引用和变量。当您为 Kubernetes 部署制定规格时，您会很容易遇到这个问题。您会发现自己在使用文字(例如，用于图像名称和摘要),甚至在受人尊敬的部署中，也在使用重复的名称和选择器。

如果你把自己局限于 YAML 和 JSON，这些问题是没有解决办法的。谷歌创建了 [Jsonnet](http://jsonnet.org/) 部分是为了解决这些问题。聪明的 Heptio 人用… [Ksonnet](http://ksonnet.io) 将 Jsonnet 扩展到了 Kubernetes。

两者都是解决上述问题(以及更多问题)的模板语言。我鼓励你考虑 Jsonnet。正如我建议考虑使用 jq 一样，学习一次 Jsonnet，并在使用 JSON 的任何地方应用它。Ksonnet 是 Kubernetes 特有的—在我有限的(！)经验——我发现从这种特殊性中获得的好处不会被学习曲线所抵消。

**C. YAML 或 JSON**

Kubernetes 对 YAML 和 JSON 几乎一视同仁。就我个人而言，我发现 YAML 更适合配置文件，因为 YAML 比 JSON 更简洁。虽然我觉得 YAML 更难写。

不过，说到理解结构和解析，我更喜欢`--output=JSON`，也因为`--output=JSONPATH`。我是 Golang 的超级粉丝，但围棋模板并不直观，我也不使用它们。

次要观点:YAML 是 JSON 的超集([链接](https://en.wikipedia.org/wiki/JSON#YAML))……等等！什么？

**D .向下狗 API 和配置**

“向下 API”给了它一个正确但同样令人困惑的名字，它是 Kubernetes 中的一个设施，通过它，pod 可以获得它们周围的集群感。我假设，正常的流程是从外部世界向上进入 Pod 和它的容器，但是有时它对容器(！)以获得关于其环境的信息，例如其节点名称|IP、Pod 的名称(空间)|IP。

向下的 API 值通过环境变量呈现给容器。环境变量用于(和其他地方一样)向容器提供配置或其他状态。使用环境变量和向下 API 实现的一个非常好的结果(有一个小警告)是容器保持与 Kubernetes 的解耦。

下面是我的好友 Sal Rashid 的一个例子，他使用向下 API 来收集节点和 Pod 状态，并将其呈现给用户:

[https://github . com/salrashid 123/istio _ hello world/blob/master/all-istio . YAML](https://github.com/salrashid123/istio_helloworld/blob/master/all-istio.yaml)

请参阅从第 76、80、84、88 行开始的部分，其中 Pod 名称、名称空间、IP 和节点名称在运行时由向下 API 提供给名为`myapp-container`的容器。

向下 API 是收集容器数据的唯一实用的方法。所以更多的是“唯一的实践”而不是“最佳实践”。

在我的许多帖子中，当我为 Kubernetes 构建解决方案时，我在本地和容器外测试流程，然后在容器中(指定环境变量)，然后在 Kubernetes 集群上。容器化的机制是一致的(见下文),即使一个通常运行在 Docker 上，一个运行在 Kubernetes 上。

在我最近写的一篇关于 Google 的容器优化操作系统的文章中，我演示了一个容器在 Docker 下本地运行，在容器优化操作系统下远程运行，然后在 Kubernetes 上运行。

这是在本地 Docker 下运行的。注意环境变量(`--env`)是如何被用来为`gcr.io/${PROJECT}/datastore`提供配置的。

```
docker run \
--interactive \
--tty \
--publish=127.0.0.1:8080:8080 \
--env=GCLOUD_DATASET_ID=${PROJECT} \
--env=GOOGLE_APPLICATION_CREDENTIALS=/tmp/${ROBOT}.key.json \
--volume=$PWD/${ROBOT}.key.json:/tmp/${ROBOT}.key.json \
gcr.io/${PROJECT}/datastore
```

这是将部署包装到容器优化的 VM 的创建中的相同结果。这次检查提供给`container-env`标志的值:

```
gcloud beta compute instances create-with-container ${INSTANCE} \
--zone=${ZONE} \
--image-family=cos-stable \
--image-project=cos-cloud \
--container-image=gcr.io/${PROJECT}/${IMAGE}@${DIGEST} \
--container-restart-policy=always \
--container-env=\
GCLOUD_DATASET_ID=${PROJECT},\
GOOGLE_APPLICATION_CREDENTIALS=/tmp/${ROBOT}.key.json \
--container-mount-host-path=\
mount-path=/tmp,\
host-path=/tmp,\
mode=rw \
--project=${PROJECT}
```

最后，这是 Kubernetes 在 YAML 的部署片段:

```
containers:
      - name: datastore
        image: gcr.io/${PROJECT}/datastore
        imagePullPolicy: Always
        volumeMounts:
          - name: datastore
            mountPath: /var/secrets/google
        env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /var/secrets/google/datastore.key.json
        - name: GCLOUD_DATASET_ID
          value: ${PROJECT}
        ports:
        - name: http
          containerPort: 8080
```

我发现在配置中使用环境变量是相当笨拙的。没有明确的特定环境变量与特定流程的有意绑定，当出现问题时，您只会意识到它们没有正确配置。在非容器化的环境中，很容易想象环境变量会发生冲突，尽管这对于容器来说是个小问题，因为如上所述，我们是在为特定的容器显式设置值。

也就是说，以这种方式使用环境变量是最佳实践。

**边斗和为什么豆荚不总是集装箱的同义词**

> 5 cl 干邑
> 2 cl 三秒
> 2 cl 柠檬汁
> 准备将所有原料倒入加满冰块的调酒器。摇匀后滤入鸡尾酒杯。

很多时候，您会创建包含单个容器的 Kubernetes Pods，并且您会奇怪为什么当您只需要一个容器时，会有 Pod 的所有开销。pod 更类似于可以运行许多容器的主机环境。很多时候，您会考虑在一个 Pod 中运行多个容器…

…并且只有一次你应该:-)

可能不止一次，但让我们坚持一次。

反模式(不要这样做)是设想您当前的配置(让我们假设一个 web 服务器和一个数据库后端),并将两者都塞进一个 Pod 中。这不是一个好主意，除非每个 web 服务器实例必须不可分割地、永远地连接到一个特定的数据库实例。这不太可能。

更有可能的是，您的 web 服务器实例应该根据*聚合*前端负载进行扩展，而您的数据库实例应该根据它们处理前端负载的*聚合*能力进行扩展(独立于此)。当您看到聚合时，请考虑服务，当您考虑服务时，请尝试设想任意数量的 pod(因为这关系到您的账单，但对于大多数其他目的，只要数量刚好满足工作负载，需要多少 pod 并不重要)。

什么时候你应该考虑每箱多个集装箱？只有当您希望补充、扩展或丰富容器中主 Pod 的行为时，这种做法才有意义。让我们回顾一下上面的 web 服务器和数据库示例。在这个场景中，希望您现在确信您将部署两个服务(和两个部署)，一个用于前端，一个用于后端。

给 web 服务器实例本身配备一个反向代理是一种很好的常见做法。通常这可能是 [Nginx](https://www.nginx.com/) 或 [HAProxy](http://www.haproxy.org/) 并且使用 [Envoy](https://www.envoyproxy.io/) 变得越来越普遍(如果你正在寻找代理，我建议考虑 Envoy；参见#F Istio)。即使你使用不同的网络服务器(例如 Apache， [Tomcat](http://tomcat.apache.org/) w/ Java 等),反向代理也能提供一致性(我们只使用例如 Envoy)。)，即使你混合了 HTTP、 [gRPC](https://grpc.io/) 、Web Sockets 等。交通。，即使您希望将一些流量导向您的网络服务器，将一些流量导向缓存(例如 [Varnish](https://varnish-cache.org/) )。

在前面的所有场景中，使用“边车”模型是有意义的。在这个模型中，主容器(您的 web 服务器)有辅助的、补充的容器(Envoy 代理、Varnish 缓存等)。).这些必须紧密耦合到特定的 web 服务器实例*和*功能上，组合就是“单元”。

很常见的是，日志、监控、跟踪和其他基础设施组件也是作为边柜交付的。这里的一个动机是分离关注点。为开发人员提供一致的需求，以产生“可管理”的代码，并为 SRE 提供选择首选工具的灵活性，因为他们知道所有代码都将记录日志、发出指标、可跟踪、一致地应用授权等。这是一种形成服务网格基础的模式(参见#F Istio)。这是最后的最佳做法，尽管是新生的做法。

**F .使用**[Istio](https://istio.io/)

小心使用 [Istio](https://istio.io/) 。

Istio(和其他服务网格)是相对新生的技术，诞生于大规模运行容器的公司(包括谷歌)。服务网格通常在每个集群的每个名称空间的每个部署中的每个 Pod 中放置一个通用代理(在 Istio 的例子中是 Envoy)。

结果是一个一致的管理基础，允许管理(我们今天将使用 Stackdriver Trace，但有迁移到 Jaeger 的计划，我们已经推出了我们的 Prometheus monitoring)和控制服务(我们知道我们所有的服务都是安全的，我们将 10%的流量路由到服务 A、B 和 C 的 canary 版本)的松散耦合。

我建议“小心”，因为这些技术是新的，有粗糙的边缘，并且正在迅速发展。但是，该方法为您提供的优势(灵活性、敏捷性、面向未来)可能远远超过成本。最重要的是，使用服务网格作为 Kubernetes 的模型，即使您还不想采用服务网格技术。

*就这些了，乡亲们！*