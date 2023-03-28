# 使用基于 GCP 日志的指标在 GKE 上部署 Flagger

> 原文：<https://medium.com/google-cloud/deployment-on-gke-with-flagger-using-gcp-log-based-metrics-9daf951348d1?source=collection_archive---------10----------------------->

## **概述**

Flagger 是一个渐进式交付工具，它将使用 Kubernetes 的应用程序的发布过程转换为自动操作。flagger 工具可用于执行部署策略(淡黄色、A/B、蓝绿色)。flagger 部署是在指标分析阶段执行的。我们可以使用基于 GCP 日志的指标来定义自定义指标，这些指标可用于控制 GKE 集群中的部署。在这篇博客中，我们将介绍如何使用 flagger 工具，使用基于 GCP 日志的指标作为 flagger 指标来执行 canary 部署。

## 先决条件

1.  安装了 Istio 并启用了工作负载标识的 GKE 集群。
2.  Flagger 需要 Kubernetes cluster **v1.16** 或更新版本，Istio **v1.5** 或更新版本。
3.  [启用 JSON 类型的 Istio 访问日志。](https://www.caseywylie.io/kubernetes/configure-istio-accesslog/)
4.  [更新 GKE 集群防火墙，为专用 GKE 集群打开端口 15017。](https://istio.io/latest/docs/setup/platform-setup/gke/?_ga=2.93334686.242401394.1668493301-1767011036.1668493301#:~:text=For%20private%20GKE%20clusters)

## 步伐

1.  在 istio-system 命名空间中使用以下命令在 GKE 集群上安装 flagger:

```
kubectl apply -k github.com/fluxcd/flagger//kustomize/istio
```

2.在测试命名空间中部署带有测试窗格的示例应用程序:

```
kubectl create ns test
kubectl label namespace test istio-injection=enabled
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main
```

3.使用入口网关公开示例应用程序:

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: public-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
```

4.使用以下云日志记录查询创建一个基于日志的分布类型指标，以从 istio 访问日志中获取示例应用程序的 canary pods(使用名为 jsonPayload.duration 的字段)的延迟:

```
resource.type="k8s_container"
resource.labels.project_id="<GCP_PROJECT_ID>"
resource.labels.location="<GKE_LOCATION>"
resource.labels.cluster_name="<GKE_CLUSTER_NAME>"
resource.labels.namespace_name="test"
resource.labels.container_name="istio-proxy"
jsonPayload.authority="podinfo-canary.test:9898"
jsonPayload.duration>0
```

5.从 GCP IAM 页面创建具有监控管理员角色的服务帐户(GCP_SA)。

6.在启用了工作量标识的 GKE 集群中执行以下步骤:

a)在 flagger 服务帐户和 GCP 服务帐户之间添加 IAM 策略绑定。此绑定允许 flagger Kubernetes 服务帐户充当 IAM 服务帐户:

```
gcloud iam service-accounts add-iam-policy-binding <GCP_SA> \    
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:<GCP_PROJECT_ID>.svc.id.goog[<NAMESPACE>/<FLAGGER_KUBERNETES_SA>]"
```

b)注释 flagger Kubernetes 服务帐户:

```
kubectl annotate serviceaccount <FLAGGER_KUBERNETES_SA> \
--namespace <NAMESPACE> \
iam.gke.io/gcp-service-account=<GCP_SA>
```

7.使用基于 GCP 日志的度量创建度量模板，以从步骤 4 中创建的基于 GCP 日志的度量中获取延迟。

```
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: canary-latency
  namespace: test
spec:
  provider:
    type: stackdriver
    secretRef: 
      name: gcloud-sa
  query: |
    fetch k8s_container
    | metric 'logging.googleapis.com/user/podinfo-canary-latency'
    | every 1m
    | group_by [], 
        [value_response_latencies_percentile:
            percentile(value.latency,99 )]
```

8.使用 gcloud 项目 ID 创建 kubernetes 秘密:

```
kubectl create secret generic gcloud-sa --from-literal=project=<project-id> -n test
```

9.应用具有以下配置的加那利 YAML 文件，如果需要，更新清单中的主机和网关参数:

```
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    # Mention the deployment name for which Canary has to be triggered
    name: podinfo
  service:
    # service port number
    port: 9898
    # container port number or name (optional)
    targetPort: 9898
    # Istio gateways (optional)
    gateways:
    - public-gateway.istio-system.svc.cluster.local
    - istio-ingressgateway
    # Istio virtual service host names (optional)
    hosts:
    # - app.example.com
    - "*"
    # Istio traffic policy (optional)
    trafficPolicy:
      tls:
        # use ISTIO_MUTUAL when mTLS is enabled
        mode: DISABLE
  analysis:
    # schedule interval (default 60s)
    interval: 30s
    # max number of failed metric checks before rollback
    threshold: 10
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 100
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    match:
      - headers:
          cookie:
            regex: "^(.*?;)?(type=insider)(;.*)?$"
    metrics:
      - name: "GCP log based metrics for canary podinfo"
        templateRef:
          name: canary-latency
          namespace: test
        thresholdRange:
          max: 100
        interval: 30s
    webhooks:
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 15s
        metadata:
          cmd: "hey -z 10m -q 10 -c 2 -H 'Cookie: type=insider' http://podinfo-canary.test:9898/"
      - name: "promotion gate"
        type: confirm-promotion
        url: http://flagger-loadtester.test/gate/approve
```

*注:* [*根据需要配置金丝雀 YAML 文件的各个参数。*](https://docs.flagger.app/usage/how-it-works)

10.通过更新示例应用程序的映像版本来触发 canary 部署:

```
kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:6.0.1
```

11.一旦金丝雀被初始化，金丝雀部署将开始，如果指标分析成功，部署的新版本将以权重 5 的增量进行部署。

12.canary 的指标分析检查基于 GCP 日志的指标的延迟是否低于新 canary 版本的阈值 100。如果度量分析条件为真，则金丝雀卷展栏会增加第 5 步。金丝雀以步长权重 5 递增，直到全部 100%的流量被路由到较新的金丝雀版本。