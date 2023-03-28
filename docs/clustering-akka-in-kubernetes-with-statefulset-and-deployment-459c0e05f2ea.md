# 使用 Statefulset 和 Deployment 对 Kubernetes 中的 Akka 进行聚类

> 原文：<https://medium.com/google-cloud/clustering-akka-in-kubernetes-with-statefulset-and-deployment-459c0e05f2ea?source=collection_archive---------0----------------------->

我在 JFokus 2017 上发言，继续看到人们对 [Akka](http://akka.io/) 的浓厚兴趣。我也想了解更多。在 JFokus speakers unconference 期间，我有机会见到了来自 Akka 团队的[约翰·安德烈](https://twitter.com/apnylle)，以及使用 Akka 和 [Lagom](https://www.lightbend.com/lagom) 的[奥拉·彼得森](https://twitter.com/gotoOla)。我趁机向他们两个学习了 Akka。

除了基础知识之外，我还对集群多个 Akka 节点以及在 Kubernetes 中如何工作非常感兴趣。一天晚饭后，约翰、奥拉和我坐在一起，开始着手这项工作。最后，我能够在 Kubernetes 建立一个 Akka 集群。你可以在 GitHub 上找到示例代码和配置:[https://github.com/saturnism/akka-kubernetes-example](https://github.com/saturnism/akka-kubernetes-example)

这篇文章抓住了我学到的东西。

# Akka 聚类基础

Akka 集群节点需要向种子节点注册自己，以便发现其他节点。种子节点需要是一组稳定有序的节点(至少 1 个，但 2 个或更多用于冗余)。每个新的 Akka 节点将尝试向列表中的第一个种子节点注册。如果种子节点顺序经常改变，那么创建裂脑集群的可能性很小。一旦群集了一组稳定的种子节点，并且随着更多的 Akka 节点加入群集，任何其他 Akka 节点都可以使用 Cluster(System)通过任何 Akka 节点以编程方式加入。加入()。

其次，Akka 节点地址(例如，Akka . TCP://cluster @ seed-node-1:2551)必须使用外部可寻址的主机名/IP。例如，如果一个 Akka 节点的 IP 地址为 10.100.0.23，主机名为 akka-23，并且它将自己注册为 Akka . TCP://cluster @ 10 . 100 . 0 . 23:2551，那么您将无法将其寻址为 Akka . TCP://cluster @ Akka-23:2551。同样，如果 Akka 节点将其自身注册为 Akka . TCP://cluster @ Akka-23:2551，那么您也不能将其地址指定为 Akka . TCP://cluster @ 10 . 100 . 0 . 23:2551。

当您在容器系统中部署并期望在主机上公开 Akka 时，这会造成一些困难，因为容器 IP 不同于主机 IP。在这种情况下，您需要使用主机名和端口配置来指定可从其他节点寻址的组合。然后，使用 bind-hostname 和 bind-port 指定本地/内部主机名和端口。例如:

```
akka {
  actor {
    provider = "cluster"
  }
  remote {
    netty.tcp {
      hostname = 192.168.0.1         # machine IP
      port = 32551                   # machine port
      bind-hostname = 10.100.0.23    # container IP
      bind-port = 2551               # container port
    }
  }
}
```

幸运的是，有了 Kubernetes 的网络架构，事情就简单多了。每个 Akka 节点都有自己的 IP 地址，任何其他 Akka 节点都可以访问该地址，即使 Akka 节点容器运行在不同的机器上。在 Kubernetes 中，您不需要处理主机/容器 IP/端口映射。

# 如何在 Kubernetes 中创建一个 Akka 集群？

我最初认为我可以采用我用来集群 Infinispan (JGroup)和 Hazelcast 的相同方法。但是考虑到上面的限制，在 Kubernetes 中创建一个可伸缩的 Akka 集群可能更具挑战性。我将研究几个选项和我的解决方案。

## 无头服务和 DNS 发现

Kubernetes 中有几个使用[无头服务](https://kubernetes.io/docs/user-guide/services/#headless-services)的 Akka 集群示例:

[](http://charithe.github.io/akka-cluster-on-kubernetes.html) [## Kubernetes 上的 Akka 集群-清醒电梦

### 运行 Akka 集群应用程序的挑战之一是发现其他节点所需的引导步骤…

charithe.github.io](http://charithe.github.io/akka-cluster-on-kubernetes.html) [](https://github.com/vyshane/klusterd/blob/master/deployment/manifests/klusterd-peers-svc.yaml) [## vyshane/klusterd

### 使用 Kubernetes 的 klusterd - Akka 聚类:一个示例项目

github.com](https://github.com/vyshane/klusterd/blob/master/deployment/manifests/klusterd-peers-svc.yaml) 

这些示例使用[无头服务](https://kubernetes.io/docs/user-guide/services/#headless-services)来创建 DNS 名称条目，例如 akka-peers。所有 Akka 节点将能够使用这个 DNS 名称条目(或 Kubernetes API)来获得集群中所有其他 Akka 节点的 IP 地址列表。所有发现的 IP 地址都可以用作种子节点。但还是要小心。默认情况下，Kubernetes headless 服务将返回循环 DNS 条目。每当您引用 akka-peers DNS 名称时，您将收到不同顺序的同一组 IP 地址:

```
root@akka-peer-ki9f:/# dig +search akka-peer
...
;; ANSWER SECTION:
akka-peer.default.svc.cluster.local. 24 IN A 10.44.3.16
akka-peer.default.svc.cluster.local. 24 IN A 10.44.2.7
akka-peer.default.svc.cluster.local. 24 IN A 10.44.0.13
akka-peer.default.svc.cluster.local. 24 IN A 10.44.3.11
...root@akka-peer-ki9f:/# dig +search akka-peer
...
;; ANSWER SECTION:
akka-peer.default.svc.cluster.local. 26 IN A 10.44.2.7
akka-peer.default.svc.cluster.local. 26 IN A 10.44.0.13
akka-peer.default.svc.cluster.local. 26 IN A 10.44.3.11
akka-peer.default.svc.cluster.local. 26 IN A 10.44.2.8
akka-peer.default.svc.cluster.local. 26 IN A 10.44.1.18
```

这对于确定 Akka 种子节点可能不起作用。此外，如果 Akka 群集中有 1，000 个 Akka 节点，初始种子节点列表可能由所有 1，000 个条目组成，而您实际上只需要其中的几个条目。

Klusterd 的工作方式是先对 IP 列表进行排序，然后只取前 5 个(见 [Klusterd 的启动脚本](https://github.com/vyshane/klusterd/blob/master/klusterd/build.sbt))。但是，随着 IP 地址的来来去去，排序后的集合可能会随着时间慢慢改变。只要原始的引导实例仍然存在，这可能是好的。

## 状态集

考虑到需要一个稳定有序的种子节点列表，我最初的想法是用[statefullset](https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/)部署整个 Akka 集群。这将允许每个 Akka 节点有一个连续和稳定的 DNS 名称。例如，第一个 akka 节点将有一个稳定的 DNS 名称 akka-0，第二个 Akka 节点将是 akka-1，依此类推。然后，我总是可以使用前两三个节点作为种子节点。我还可以通过增加副本的数量来进行横向扩展。

但是，状态集是按顺序缩放的，而不是并行缩放的。当我将一个 Statefulset 从一个 Akka 节点扩展到五个 Akka 节点时，它将启动 akka-1，等待它成功启动，然后启动 akka-2，等待，启动 akka-3，…并重复，直到完成。这并不理想，因为启动数百个 Akka 节点可能需要一段时间。

> 注 8/10/2017:本文是为 Kubernetes 1.6 编写的。从 Kubernetes 1.7 开始，StatefulSet 可以并行启动[多个实例](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/#parallel-pod-management)。我还没有测试过，但是如果你使用 Kubernetes 1.7，它可能是比下面描述的混合解决方案更好的选择。

## 混合解决方案

我最后的想法是同时使用 Kubernetes 的 Statefulset 和 Deployment。我可以对种子节点使用 Statefulset，对工作节点使用 Deployment。通过这种方式，我可以为种子节点维护稳定有序的 DNS 名称，同时在随部署扩展时，我还能并行启动多个 Akka 节点。

完整的例子在我的 GitHub 上:

[](https://github.com/saturnism/akka-kubernetes-example) [## 农神主义/阿卡-库伯内特斯-示例

### 在 GitHub 上创建一个帐户，为 akka-kubernetes-example 开发做出贡献。

github.com](https://github.com/saturnism/akka-kubernetes-example) 

为了对种子节点使用 Statefulset，我首先创建了一个无头服务。资源定义看起来像常规服务，但 clusterIP 设置为 None:

```
apiVersion: v1
kind: Service
metadata:
  name: akka-seed
spec:
  ports:
  - port: 2551
    protocol: TCP
    targetPort: 2551
  selector:
    run: akka-seed
  clusterIP: None
```

Statefulset 中的每个 Akka 种子节点实例都可以通过 DNS 作为＄{ SEED _ NODE _ NAME }进行寻址。阿卡种子。

然后，我创建一组有状态的种子节点:

```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  labels:
    run: akka-seed
  name: akka-seed
spec:
  serviceName: akka-seed
  replicas: 2
  selector:
    matchLabels:
      run: akka-seed
  template:
    metadata:
      labels:
        run: akka-seed
    spec:
      containers:
      - name: akka-seed
        image: saturnism/akka-cluster-example
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: SEED_NODES
          value: akka-seed-0.akka-seed,akka-seed-1.akka-seed
        command: ["/bin/sh", "-c", "HOST_NAME=${POD_NAME}.akka-seed java -jar /app/app.jar"]
        livenessProbe:
          tcpSocket:
            port: 2551
        ports:
        - containerPort: 2551
          protocol: TCP
```

首先，每个种子节点将具有稳定的名称(例如，akka-seed-0、akka-seed-1 等)。这个名字被称为 Kubernetes 豆荚的名字。我使用了[Kubernetes download API](https://kubernetes.io/docs/user-guide/downward-api/)将 Pod 名称公开为环境变量，然后用它来构造可寻址的 DNS 名称，比如 akka-seed-0.akka-seed。

对于 SEED_NODE，我可以简单地给它每个种子节点的稳定 DNS 名称。

要部署 Akka 种子节点:

```
$ kubectl apply -f akka-seeds.yaml
```

看看依次产生的阿卡种子:

```
$ kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
akka-seed-0                   1/1       Running   0          15h
akka-seed-1                   1/1       Running   0          15h
```

接下来，我可以使用 Deployment 来部署工作节点。它还使用向下 API 将 Pod IP 地址分配为 Akka 节点的主机名。

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  ...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: akka-worker
        image: saturnism/akka-cluster-example
        env:
        - name: SEED_NODES
          value: akka-seed-0.akka-seed,akka-seed-1.akka-seed
        - name: HOST_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        image: saturnism/akka-cluster-example
        ...
```

部署后，您可以验证所有节点都已启动并正在运行:

```
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
akka-seed-0                    1/1       Running   0          8s
akka-seed-1                    1/1       Running   0          6s
akka-worker-2263404214-8c266   1/1       Running   0          8s
akka-worker-2263404214-9ws3k   1/1       Running   0          8s
akka-worker-2263404214-f2tp3   1/1       Running   0          8s
akka-worker-2263404214-lkvz3   1/1       Running   0          8s
```

您可以通过检查任何 Akka 节点的日志来验证节点是否已经加入集群。例如，要跟踪第一个种子节点的日志:

```
$ kubect logs -f akka-seed-0
[INFO] [02/12/2017 15:36:53.568] [main] [akka.remote.Remoting] Starting remoting
[INFO] [02/12/2017 15:36:53.707] [main] [akka.remote.Remoting] Remoting started; listening on addresses :[akka.tcp://ClusterSystem@akka-seed-0.akka-seed:2551]
...
[INFO] [02/12/2017 15:37:05.101] [ClusterSystem-akka.actor.default-dispatcher-16] [akka.cluster.Cluster(akka://ClusterSystem)] Cluster Node [akka.tcp://ClusterSystem@akka-seed-0.akka-seed:2551] - Node [akka.tcp://ClusterSystem@akka-seed-1.akka-seed:2551] is JOINING, roles []
[INFO] [02/12/2017 15:37:06.854] [ClusterSystem-akka.actor.default-dispatcher-16] [akka://ClusterSystem/user/$a] Member is Up: Member(address = akka.tcp://ClusterSystem@10.44.2.10:2551, status = Up)
...
```

要取消部署所有内容，只需删除服务、状态集和部署。或者，一步到位:

```
$ kubectl delete -f kubernetes/
```

# 试试看！

你可以在 GitHub 上找到代码和配置。您可以在任何 Kubernetes 1.5 集群中尝试这样做。启动多节点集群最简单的方法之一是在 [Google Container Engine](https://cloud.google.com/container-engine/) 上，但是从一个节点开始使用 [Minikube](https://github.com/kubernetes/minikube) 对于本地开发来说非常好。

特别感谢[约翰](https://twitter.com/apnylle)和[奥拉](https://twitter.com/gotoOla)帮助学习 Akka 和评论这篇文章。