# IP 地址管理策略——运行 GKE 的一个重要方面

> 原文：<https://medium.com/google-cloud/ip-address-management-strategy-a-crucial-aspect-of-running-gke-f063fe90cfbd?source=collection_archive---------0----------------------->

![](img/a1905ccc5478e588b3d17ae6fe0623e0.png)

*您的组织是否在为每个应用程序/环境构建新的 GKE 实例？*
*您的应用团队是否要求您为 pod 提供/18 或/16 或/14 子网 CIDRs？*
*pod 与您的内部应用程序通信吗？*

**那么你很快就会遇到 IP 枯竭的问题。不这么认为？让我告诉你怎么做！**

任何 Kubernetes 网络模型都严重依赖 IP 地址。
服务、pod、容器和节点使用 IP 地址和端口进行通信。

当您在 VPC 中启动 GKE 集群时，需要以下子网:

*   您的工作节点的主要子网范围
*   pod 和服务的二级范围([别名 IP](https://cloud.google.com/vpc/docs/alias-ip) 范围)
*   控制平面 CIDR 范围—用于私有集群中的主控制平面节点 IP 地址

让我们举个例子来更好地理解这一点:

## **应用要求:**

*   微服务数量:50
*   每个微服务需要 0.5 个 CPU、1GB 内存和最少 3 到最多 10 个 pod

## **我们如何确定 GKE 集群的规模？**

我们将继续使用*区域标准 GKE 集群*，因为它提供了高可用性和分区故障保护。为什么区域集群更好？在这里阅读更多。

我们希望部署的地区是“亚洲-东南亚 1”

**所需资源总额:**

*   最小 CPU:0.5 x 50 x 3 = 75；最小内存:1 x 50 x 3 = 150 GB
*   最大 CPU:0.5 x 50 x 10 = 250；最大内存:1 x 50 x 10 = 500 GB

最小集群大小将为:

*   每个节点大小:8 个 vCPU 16 GB RAM
*   节点数量:12
*   节点类型:N2

我们将启用节点自动配置(NAP)来优化随着负载增加的节点自动扩展(有关 NAP 的更多详细信息，请参考[文档](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning)):

大约。最大群集大小可能是:

*   每个节点大小:6 个 vCPU 12 GB RAM
*   节点数量:24
*   节点类型:N2

因此，GKE 集群所需的子网如下:

*   工作节点的主要子网范围: <rfc ip="" address="" range="">/24</rfc>
*   Pod 的二级范围(别名 IP 范围): <rfc ip="" address="" range="">/18</rfc>
*   服务的二级范围(别名 IP 范围): <rfc ip="" address="" range="">/20</rfc>
*   控制飞机 CIDR 范围- <rfc ip="" address="" range="">/28</rfc>

*(很快会写一篇详细的博客解释为什么选择上面的 IP 子网)*

## 问题是:

在 Google Cloud 中，别名 IP 范围是可路由的，也就是说，您可以从 VPC 内部使用集群中的 IP 直接呼叫 pod 分配给作为集群 IP 类型服务公开的 k8 服务的 IP 地址仍然只能在集群内部路由]。因此，如果 pod 必须与内部网络通信，IP 地址可能会重叠。
因此，网络团队需要在分配前规划 CIDR，并且需要了解整个组织中已使用/未使用的 CIDR 块。

这就是问题所在，从您的数据中心保留每个集群所需的/14 /16 /18 CIDR 范围并不是最佳选择。

## **解决方案:**

为了解决超出 IP 范围的问题，我们建议如下:

**RFC 对非 RFC** :

*   从 **RFC 1918 范围**中为工作节点和控制平面 CIDR 范围选择您的主要子网范围
*   从 [**非 RFC 1918 范围**](https://cloud.google.com/vpc/docs/vpc#valid-ranges) 中为 pod 和服务选择辅助子网范围

由于这两个范围(RFC 和非 RFC)仍然会被云路由器暴露给您的内部数据中心，因此我们使用 [*IP 伪装代理*](https://cloud.google.com/kubernetes-engine/docs/how-to/ip-masquerade-agent) 到[*SNAT*](https://en.wikipedia.org/wiki/Network_address_translation#SNAT)*pod IP 范围和节点 IP。*这将有助于将从 Pods (SNAT)发送的出站数据包的源 IP 地址更改为节点 IP 地址。

**要在您的 GKE 集群中启用此功能，请执行以下步骤:**

> 检查 ip-masq 代理是否已经安装在集群中:

```
kubectl get daemonsets/ip-masq-agent -n kube-system
```

如果 ip-masq-agent DaemonSet 存在，则输出类似于以下内容:

```
NAME    DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE SELECTOR AGEip-masq-agent 3  3    3       3          3         <none>    13d
```

如果 ip-masq-agent DaemonSet 不存在，则输出如下所示:

```
Error from server (NotFound): daemonsets.apps “ip-masq-agent” not found
```

> 使用以下内容创建一个文件 ipmasq-cm.yaml:

```
apiVersion: v1
data:
  config: |
    nonMasqueradeCIDRs:
      - CIDR_1
      - CIDR_2
    masqLinkLocal: false
    resyncInterval: SYNC_INTERVAL
kind: ConfigMap
metadata:
  name: ip-masq-agent
  namespace: kube-system
```

CIDR_1 和 CIDR_2 是你不希望 SNAT 发生的目的地的 CIDR。对于每个其他目的地，pod IP 将在出口处被替换为节点 IP 作为源。

SYNC_INTERVAL 间隔时间，在此时间之后，pod 将与配置同步。

> 在集群中部署文件

```
kubectl create configmap ipmasq-cm --namespace=kube-system --from-file=config=ipmasq-cm.yaml
```

> 创建一个 ipmasq-agent daemon set“ipmasq-agent . YAML”并在集群中部署:

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ip-masq-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: ip-masq-agent
  template:
    metadata:
      labels:
        k8s-app: ip-masq-agent
    spec:
      hostNetwork: true
      containers:
      - name: ip-masq-agent
        image: k8s.gcr.io/networking/ip-masq-agent:v2.7.0
        args:
            # The masq-chain must be IP-MASQ
            - --masq-chain=IP-MASQ
            # To non-masquerade reserved IP ranges by default,
            # uncomment the following line.
            # - --nomasq-all-reserved-ranges
        securityContext:
          privileged: true
        volumeMounts:
          - name: config-volume
            mountPath: /etc/config
      volumes:
        - name: config-volume
          configMap:
            name: ip-masq-agent
            optional: true
            items:
              - key: config
                path: ip-masq-agent
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      - key: "CriticalAddonsOnly"
        operator: "Exists"
```

有关配置 IP 伪装代理的更多信息，请参考 GKE [公共文档](https://cloud.google.com/kubernetes-engine/docs/how-to/ip-masquerade-agent)。

## 要记住的事情:

1.  Windows 服务器节点池不支持此功能
2.  不适用于规格为**的容器中的容器。主机网络:真**
3.  群集的 Pod IP 地址范围不应匹配，或者不在 10.0.0.0/8 范围内
4.  对于自动驾驶集群，需要部署出口 NAT 策略。

阅读更多关于在 GKE 选择替代网络模型的 [GKE IP 管理策略](https://cloud.google.com/architecture/gke-ip-address-mgmt-strategies)的信息，或者随时联系我！

请关注[谷歌云社区](https://medium.com/google-cloud)，了解更多此类有见地的博客。黑客快乐！