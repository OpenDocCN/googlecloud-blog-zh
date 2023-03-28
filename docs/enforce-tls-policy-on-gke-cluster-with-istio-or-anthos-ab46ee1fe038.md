# 使用 Istio 或 Anthos 在 GKE 集群上实施 TLS 策略

> 原文：<https://medium.com/google-cloud/enforce-tls-policy-on-gke-cluster-with-istio-or-anthos-ab46ee1fe038?source=collection_archive---------0----------------------->

![](img/daaf3c516abdcba89d12c5e3544143f1.png)

照片由[乔恩·摩尔](https://unsplash.com/@thejmoore?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)在 [Unsplash](https://unsplash.com/s/photos/security?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) 拍摄

Google Cloud 提供了一种[简单的方式](https://cloud.google.com/load-balancing/docs/use-ssl-policies)来强制执行 TLS 策略，例如 SSL 和 TLS 协议的允许版本以及允许用于这些版本的密码套件。可以在 [HTTP(S)外部负载均衡器](https://cloud.google.com/load-balancing/docs/https)进行配置。如果您的工作负载运行在不使用 Google 负载平衡器的方式公开服务的 GKE 集群上，或者工作负载是内部公开的(例如，使用 HTTP(S)内部负载平衡器)，那么您需要另一种方式来实施 TLS 策略。下面的教程展示了如何使用 [Anthos 服务网格](https://cloud.google.com/service-mesh/docs/overview) (ASM)服务对其进行配置。也可以使用 Istio OSS 实现教程。但是，ASM 允许将 TLS 策略应用于入口和 pod 到 pod 的流量，而 Istio 仅支持入口流量的 TLS 策略。

# 设置环境

该教程在 GCP 上运行，需要一个有效的计费帐户的项目。本教程中的命令意味着有一个环境变量`PROJECT_ID`存储了该项目的项目 id。

> **注意:**如果您运行本教程，您将因使用 GCP 资源而付费。启用某些 API 可能会产生额外的成本。

# 1.启用 Google APIs

使用以下命令启用所需的 API:

```
# gcloud services enable — project=$PROJECT_ID \
    compute.googleapis.com container.googleapis.com \
    meshtelemetry.googleapis.com meshconfig.googleapis.com \
    iamcredentials.googleapis.com gkeconnect.googleapis.com \
    gkehub.googleapis.com cloudresourcemanager.googleapis.com \
    meshca.googleapis.com
```

如果你使用 Istio OSS，你只需要`compute.googleapis.com`和`container.googleapis.com`API。

# 2.设置 GKE 集群

带有 ASM 的 GKE 集群至少需要 e2-standard-4 机器类型。本教程没有在由较小机器组成的集群上进行测试。创建 GKE 集群:

```
# gcloud container clusters create microservice-demo \
    --zone us-central1-a \
    --machine-type=e2-standard-4 \
    --num-nodes=4 \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --no-enable-autoupgrade \
    --no-enable-autorepair \
    --tags=microservice-demo \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --project=$PROJECT_ID
# gcloud container clusters get-credentials microservice-demo \
    --zone us-central1-a --project=$PROJECT_ID
```

**注意:**如果您打算使用 Istio OSS，则无需为集群提供`--workload-pool`参数。

# 3.设置 ASM

从 ASM 1.9 开始，支持强制执行 pod 到 pod TLS 策略。以下命令安装 ASM 1.9 的最新版本:

```
# curl [https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9](https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9) > install_asm
# curl [https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9.sha256](https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9.sha256) > install_asm.sha256
# sha256sum -c --ignore-missing install_asm.sha256
# chmod +x install_asm
# ./install_asm --project_id $PROJECT_ID \
    --cluster_name microservice-demo \
    --cluster_location us-central1-a --enable_all \
    -o envoy-access-log -D asm_install --mode install
```

需要获取 ASM 版本以备后用:

```
ASM_REVISION=$(kubectl get pod -n istio-system -l app=istiod \
  -o jsonpath='{.items[0].metadata.labels.istio\.io/rev}')
```

# 部署演示应用程序并配置 TLS 策略

该教程使用了一个在线精品店应用程序，可以在 [github](https://github.com/GoogleCloudPlatform/microservices-demo) 上找到。以下步骤将应用程序部署到专用命名空间中，使用正确的证书配置 ASM 入口网关，然后对入口和 pod 到 pod 流量应用 TLS 策略。

# 1.设置一个演示命名空间来运行演示应用程序

配置一个 Kubernetes 名称空间，在其中运行演示应用程序。

```
# kubectl create ns demo
# kubectl label ns demo \
    istio-injection- istio.io/rev=$ASM_REVISION --overwrite
```

对部署到演示命名空间中的所有工作负载实施 MTL:

```
# cat <<EOF | kubectl apply -f -
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: demo
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF
```

# 2.下载并部署演示应用程序

在线精品店是在 GKE 上运行的微服务应用程序，用于不同的演示目的。本教程使用最新版本的微服务容器映像。

```
# git clone [https://github.com/GoogleCloudPlatform/microservices-demo.git](https://github.com/GoogleCloudPlatform/microservices-demo.git)
# kubectl apply -n demo \
    -f microservices-demo/release/kubernetes-manifests.yaml
# kubectl apply -n demo \
    -f microservices-demo/release/istio-manifests.yaml
```

# 3.为演示应用程序选择一个主机名并设置 DNS

要测试 TLS 的工作，您需要有一个演示应用程序的有效域名。有几种方法可以配置它。您可以使用云 DNS 或其他 DNS 服务，如您的域注册商提供的服务。您可以使用本地/etc/hostname 来进行测试。无论采用哪种方式，都可以获得 ASM 网关的 IP 地址:

```
# HOST_IP=$(kubectl get svc/istio-ingressgateway -n istio-system \
    -o=jsonpath={.status.loadBalancer.ingress[0].ip})
```

并将其与您选择的主机名相关联。对于以下步骤，将使用主机名`demo.acme.cloud` demo.acme.cloud。请将其替换为您选择的主机名。

# 4.对入口流量实施 TLS 策略

要对入口流量实施 TLS 策略，您必须配置 ASM 入口网关。为此，必须为演示应用主机名配置入口网关(更多详细信息，请参见[安全网关](https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host))。

## 4.1 为 acme.cloud 生成 CA 证书:

```
# openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
    -subj '/O=Botique Inc./CN=mydomain.com' \
    -keyout mydomain.com.key \
    -out mydomain.com.crt
```

## 4.2 生成由 CA 签名的 demo.acme.cloud 主机证书:

```
# openssl req -out demo.acme.cloud.csr \
    -newkey rsa:2048 -nodes \
    -keyout demo.acme.cloud.key \
    -subj "/CN=demo.acme.cloud/O=Demo Org"
# openssl x509 -req -days 365 -set_serial 0 \
    -CA acme.cloud.crt -CAkey acme.cloud.key \
    -in demo.acme.cloud.csr \
    -out demo.acme.cloud.crt
```

## 4.3 创建 TLS 机密:

```
# kubectl create -n istio-system secret tls ingress-credential \
    --key=demo.acme.cloud.key \
    --cert=demo.acme.cloud.crt
```

## 4.4 更新虚拟服务:

编辑虚拟服务:

```
# EDITOR=vim; kubectl edit virtualservice/frontend-ingress -n demo
```

并将`hosts:`列表中的`'*'`全部替换为`'demo.acme.cloud'`。

## 4.5 更新 Istio 网关:

编辑前端网关:

```
# EDITOR=vim; kubectl edit gateway/frontend-gateway -n demo
```

并将`servers:`下的**所有**数据替换为:

```
- hosts:
    - demo.acme.cloud
    port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: ingress-credential # must be the same as secret## the following lines define TLS policy
      maxProtocolVersion: TLSV1_3
      minProtocolVersion: TLSV1_2
      cipherSuites:
      - ECDHE-RSA-AES256-GCM-SHA384
      - AES256-GCM-SHA384
      - TLS_AES_256_GCM_SHA384
```

**注意:**保留缩进-检查以保留清单的 YAML 格式。

网关清单的`tls`部分将 TLS 策略定义为一系列 TLS 协议版本和这些版本允许的一组密码套件。注意，根据 TLS 1.3 协议规范，TLS 1.3 的密码套件不能改变。因此这组密码套件适用于≤ TLS 1.2 的协议版本。

# 5.对单元到单元的通信实施 TLS 策略

到目前为止，所有步骤都可以使用 Istio OSS 运行。当启用 mTLS 时，可以使用 ASM 强制执行 pod 到 pod 通信的 TLS 策略。与可以在每个主机的基础上启用和配置的用于入口流量的 TLS 策略相反，用于 pod 到 pod 通信的 TLS 策略是以集群方式启用的。为此，编辑 ASM 控制器:

```
# EDITOR=vim; kubectl edit deploy/istiod-$ASM_REVISION \
     -n istio-system
```

并将以下环境变量添加到部署的单个容器规范中:

```
- name: TLS_INBOUND_CIPHER_PROFILE
  value: custom
- name: TLS_INBOUND_CIPHER_SUITES
  value: ECDHE-RSA-AES256-GCM-SHA384,AES256-GCM-SHA384
```

默认情况下，ASM 仅将 TLS 协议版本限制为 1.2 和 1.3。因此，没有必要限制 TLS 协议版本的范围。

# 包裹

可以限制哪些 TLS 协议版本和哪些密码套件将用于 GKE 内的入口和点对点通信。这些值可以使用 Istio OSS 或 ASM 网关进行入口配置，使用 ASM 控制器进行 pod-to-pod 通信。